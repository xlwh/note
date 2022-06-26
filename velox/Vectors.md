# Vectors

Velox vector是列式内存格式，用于在执行期间存储查询数据。 它与 Arrow 类似，但具有更多的编码和不同的字符串、数组和映射布局，支持乱序写入。 vector构成了 Velox 库的基础。

Velox vector支持标量和复杂类型，并有几种不同的编码:

- Flat
- Constant
- Dictionary
- Bias
- Sequence

在本指南中，我们将讨论Flat、Constant和Dictionary编码。 Bias和Sequence编码不在本指南的范围内。

支持的类型有：

- Scalar types:
  - BOOLEAN
  - TINYINT
  - SMALLINT
  - INTEGER
  - BIGINT
  - TIMESTAMP
  - REAL
  - DOUBLE
  - DECIMAL
  - VARCHAR
  - VARBINARY
  - OPAQUE*
- Complex types:
  - ARRAY
  - MAP
  - ROW

单个vector表示单列的多行。 RowVector 用于表示多列的一组行以及结构类型的单列的一组行。

每个vector都有一个类型、编码和大小。 向量大小是存储在向量中的行数。 Flat、常量和字典编码可以与任何类型组合。



## Buffer

Vector数据使用一个或多个缓冲区存储在内存中。 缓冲区是一个连续的字节范围。 缓冲区是引用计数的，并且必须由 BufferPtr 持有。 缓冲区既可以拥有自己的内存，也可以代表外部管理内存（BufferView）的视图。 拥有其内存的缓冲区 (AlignedBuffer) 从 MemoryPool 分配内存。 如果有一个对它的引用，那么这样的缓冲区可以处于可变状态，例如 缓冲区是唯一引用的。 如果有多个对缓冲区的引用，则该缓冲区是只读的。

缓冲区本身没有类型，但它们提供特定于类型的转换方法。

AlignedBuffer::allocate<T>(size, pool) 静态方法分配一个缓冲区，该缓冲区至少可以容纳来自 MemoryPool 的 T 类型的“大小”值。

分配一个缓冲区来存储 100 个 64 位整数：

`BufferPtr values = AlignedBuffer::allocate<int64_t>(100, pool);`

结果缓冲区的长度至少为 sizeof(int64_t) * 100 = 800 字节。 您可以访问原始缓冲区以通过 as<T>() 模板方法读取和使用 asMutable<T>() 模板方法进行写入：

```
// Read-only access.
const int64_t* rawValues = values->as<int64_t>();

// Read-write access.
int64_t* rawValues = values->asMutable<int64_t>();
```

分配一个缓冲区来存储 100 个布尔标志：

```
BufferPtr flags = AlignedBuffer::allocate<bool>(100, pool);
```

结果缓冲区的长度至少为 100 / 8 = 13 字节（Velox 对每个布尔标志仅使用 1 位）。 与其他类型不同，布尔值不能通过 bool 指针单独访问。 相反，您将缓冲区解释为无符号 64 位整数的连续数组，并使用“位”命名空间中的辅助方法访问各个值。

```
// Read-only access.
const uint64_t* rawFlags = flags->as<uint64_t>();

// Check if flag # 12 is set.
bool active = bits::isBitSet(rawFlags, 12);

// Read-write access.
uint64_t* rawFlags = flags->asMutable<int64_t>();

// Set flag 15 to true/on.
bits::setBit(rawFlags, 15);

// or
bits::setBit(rawFlags, 15, true);

// Set flag 16 to false/off.
bits::clearBit(rawFlags, 16);

// or
bits::setBit(rawFlags, 16, false);
```

由于缓冲区没有类型，缓冲区的大小反映了缓冲区中的字节数，而不是值的数量。



## NullFlags

每个vector都包含一组可选的NullFlags，用于标识具有空值的行。 如果给定向量中没有空值，则可能不存在空标志。 空标志位打包成一个 64 位无符号整数数组。 零表示空值。 一表示非空值。 （这种违反直觉的选择是出于与 Arrow 的兼容性。）

在下图中，位置 2、7 和 11 为空。

![../_images/nulls.png](https://facebookincubator.github.io/velox/_images/nulls.png)

BaseVector 是各种向量的基类。 它包含存储在向量中的值的类型，例如 INTEGER 与 VARCHAR、空值缓冲区和向量中的行数。

```
std::shared_ptr<const Type> type_;
BufferPtr nulls_;
vector_size_t length_ = 0;
```

向量始终由 std::shared_ptr 使用 VectorPtr 别名保存。

`using VectorPtr = std::shared_ptr<BaseVector>;`

“bits”命名空间包含许多使用空缓冲区的转换函数。

```
// Check if position #12 is null.
bool isNull = bits::isBitNull(rawNulls, 12);

// Set position #12 to null.
bits::setNull(rawNulls, 12);
bits::setNull(rawNulls, 12, true);

// Set position #12 to non-null.
bits::clearNull(rawNulls, 12);
bits::setNull(rawNulls, 12, false);
```



## Flat Vectors - Scalar Types

Flat vectors使用 FlatVector<T> 模板表示，其中 T 是存储在向量中的值的类型。 这些类型是：

| Velox Type | C++ Type          | Bytes per Value    |
| :--------- | :---------------- | :----------------- |
| BOOLEAN    | bool              | 0.125 (e.g. 1 bit) |
| TINYINT    | int8_t            | 1                  |
| SMALLINT   | int16_t           | 2                  |
| INTEGER    | int32_t           | 4                  |
| BIGINT     | int64_t           | 8                  |
| REAL       | float             | 4                  |
| DOUBLE     | double            | 8                  |
| TIMESTAMP  | struct Timestamp  | 16                 |
| VARCHAR    | struct StringView | 16                 |
| VARBINARY  | struct StringView | 16                 |

FlatVector<T> 包含一个值缓冲区，如果 T = StringView 则包含一个或多个字符串缓冲区。 值缓冲区是一个连续的字节缓冲区，每个值有 sizeof(T) 个字节，包括空值。 每个值的字节数对于不同的类型是不同的。 布尔值是位压缩的，每 8 个值使用 1 个字节。

```
BufferPtr values_;
std::vector<BufferPtr> stringBuffers_;
```

FlatVector<T> 使用 BaseVector 作为基类，并从中获取类型、大小和空值缓冲区。 与所有其他向量一样，平面向量由 std::shared_ptr 使用 FlatVectorPtr 别名保存。

```
template <typename T>
using FlatVectorPtr = std::shared_ptr<FlatVector<T>>;
```

下图显示了一个具有 11 个值的 INTEGER 类型的平面向量。 该向量表示为 FlatVector<int32_t>。 values_ 缓冲区至少有 11 个连续条目的空间，每个条目 4 个字节。 Nulls 缓冲区至少有 11 个连续条目的空间，每个条目有 1 位。 位置 2,7, 11 的值为空，例如 nulls_buffer 中的位 2、7、11 为 0。nulls_buffer 中的其余位为 1。 values_buffer 中的条目 2、7、11 包含垃圾。

![../_images/flat-vector.png](https://facebookincubator.github.io/velox/_images/flat-vector.png)

包括字符串在内的所有标量值都是固定长度的，例如 每个值都存储在固定数量的字节中。 由于字符串可以是可变长度的，因此实际的字符串存储在一组与值缓冲区分开的字符串缓冲区中。 值缓冲区存储 16 字节 StringViews，其中包括 4 字节字符串大小、4 字节前缀和指向字符串缓冲区之一中完整字符串的 8 字节指针。 最多 12 个字符的短字符串完全存储在 StringView 结构中。 它们占据了被前缀和指针过度占用的空间。



![../_images/string-view-layout.png](https://facebookincubator.github.io/velox/_images/string-view-layout.png)

下图说明了长字符串和短字符串在内存中表示的差异。 “黄石国家公园”是一个 25 个字符的长字符串，太长而无法内联。 因此，StringView 在字符串缓冲区中存储了一个 4 字节前缀“Yell”和一个指向整个字符串的指针。 “大雨”字符串只有 10 个字符长，因此内联存储在 StringView 中。 将长字符串的前缀存储在 StringView 中可以优化比较操作。

![../_images/string-views.png](https://facebookincubator.github.io/velox/_images/string-views.png)

字符串缓冲区中的字符串不一定按顺序显示，各个字符串之间可能存在间隙。 单个向量可以使用一个或多个字符串缓冲区。

下图显示了一个具有 7 个值的 VARCHAR 类型的向量。 此向量表示为 FlatVector<StringView>。 values_buffer 至少有 7 个条目的空间，每个条目 16 个字节。 stringBuffers_ 数组有一个包含非内联字符串连接的条目。 values_buffer 中的每个条目使用 4 个字节来存储字符串的大小。

![../_images/string-vector.png](https://facebookincubator.github.io/velox/_images/string-vector.png)

固定宽度值允许无序填充向量，例如 在为第 2 行写入值之前为第 5 行写入值。这在执行条件表达式时很有用。

注意：任何类型（标量或复数）的 Velox 向量都可以乱序写入。 这是 Velox 向量和箭头数组之间的主要区别。

允许字符串缓冲区中的字符串出现乱序并在各个字符串之间出现间隙，从而可以实现 substr 和 split 等函数的零拷贝实现。 这些函数的结果由原始字符串的子字符串组成，因此可以使用指向与输入向量相同的字符串缓冲区的 StringView。

将 substr(s, 2) 函数应用于上面显示的向量的结果如下所示：

![../_images/substr-result.png](https://facebookincubator.github.io/velox/_images/substr-result.png)

该向量使用与原始向量相同的字符串缓冲区。 它只是使用 std::shared_ptr 引用它。 各个 StringView 条目要么包含内联字符串，要么引用原始字符串缓冲区中的位置。 在位置 1 应用 substr(s, 2) 函数字符串后，字符串变得足够短以适合 StringView，因此，它不再包含指向字符串缓冲区中某个位置的指针。

TIMESTAMP 类型的平面向量由 FlatVector<Timestamp> 表示。 Timestamp 结构由两个 64 位整数组成：秒和纳秒。 每个条目使用 16 个字节。

```
int64_t seconds_;
uint64_t nanos_;
```

## Constant Vector - Scalar Types

常量向量使用 ConstantVector<T> 模板表示，该模板包含一个值和一个布尔值，指示该值是否为空。 如果 T = StringView 并且字符串长度超过 12 个字符，它可能包含一个字符串缓冲区。

```
T value_;
bool isNull_ = false;
BufferPtr stringBuffer_;
```

BaseVector::wrapInConstant() 静态方法可用于从标量值创建常量向量。

```
static std::shared_ptr<BaseVector> createConstant(
    variant value,
    vector_size_t size,
    velox::memory::MemoryPool* pool);
```

## Dictionary Vector - Scalar Types

字典编码用于紧凑地表示具有大量重复值的向量以及过滤器或类似过滤器操作的结果，而无需复制数据。 字典编码也可用于表示排序操作的结果，而无需复制数据。

标量类型的字典向量由 DictionaryVector<T> 模板表示。 字典向量包含一个指向基向量的共享指针，该指针可能是平面的，也可能不是平面的，以及一个索引缓冲区到基向量中。 索引是 32 位整数。

```
BufferPtr indices_;
VectorPtr dictionaryValues_;
```

这是一个表示颜色的 VARCHAR 类型的字典向量。 基本向量仅包含 5 个条目：红色、蓝色、黄色、粉色、紫色和金色。 字典向量包含一个指向基向量的 std::shared_ptr 以及一个指向该向量的索引缓冲区。 字典向量中的每个条目都指向基向量中的一个条目。 条目 0 和 2 都指向包含“red”的基向量中的条目 0。 条目 1、4、5、10 指向相同但不同的条目 1，其中包含“蓝色”。 这种编码避免了复制重复的字符串。

![../_images/dictionary-repeated2.png](https://facebookincubator.github.io/velox/_images/dictionary-repeated2.png)

多个字典向量可以引用同一个基向量。 我们说字典向量包装了基向量。

这是一个表示过滤器结果的 INTEGER 类型的字典：n % 2 = 0。基向量包含 11 个条目。 这些条目中只有 5 个通过了过滤器，因此字典向量的大小为 5。索引缓冲区包含 5 个条目，这些条目引用了通过过滤器的原始向量中的位置。

![../_images/dictionary-subset2.png](https://facebookincubator.github.io/velox/_images/dictionary-subset2.png)

当过滤器或类似过滤器的操作应用于多个列时，结果可以表示为多个字典向量，它们都共享相同的索引缓冲区。 这允许减少索引缓冲区所需的内存量，并通过剥离共享字典实现有效的表达式执行。

字典编码用于表示连接的结果，其中探测侧列被包装到字典中，以避免在构建侧复制具有多个匹配项的行。 字典编码也用于表示未嵌套的结果。

字典向量可以包装任何其他向量，包括另一个字典。 因此，可以在单个向量之上拥有多层字典，例如 Dict(Dict(Dict(Flat))).



BaseVector::wrapInDictionary() 静态方法可用于将任何给定向量包装在字典中。

`static std::shared_ptr<BaseVector> wrapInDictionary(`

`BufferPtr nulls, BufferPtr indices, vector_size_t size, std::shared_ptr<BaseVector> vector);`



BaseVector 中定义的wrappedVector() 虚拟方法提供对字典最内层向量的访问，例如 Dict(Dict(Flat))->wrappedVector() 返回 Flat。

WrapIndex(index) 在 BaseVector 中定义的虚拟方法将索引转换为字典向量到最内层向量的索引，例如 WrappedIndex(3) 为上面的字典向量返回 6。

字典向量有自己的空值缓冲区，独立于基向量的空值缓冲区。 即使基向量没有空值，这也允许字典向量表示空值。 我们说“字典包装将空值添加到基向量中”。



这是一个例子。 字典中的条目#4 被标记为空。 索引缓冲区中的相应条目包含垃圾，不应访问。

![../_images/dictionary-with-nulls.png](https://facebookincubator.github.io/velox/_images/dictionary-with-nulls.png)

## Flat Vectors - Complex Types

### Arrays

复杂类型 ARRAY、MAP 和 ROW / STRUCT 的平面向量使用 ArrayVector、MapVector 和 RowVector 表示。

ArrayVector 存储类型为 ARRAY 的值。 除了空缓冲区之外，它还包含偏移量和大小缓冲区以及元素向量。 偏移量和大小是 32 位整数。

```
BufferPtr offsets_;
BufferPtr sizes_;
VectorPtr elements_;
```

Elements 向量包含所有数组的所有单个元素。 特定数组中的元素按顺序彼此相邻出现。 每个数组条目都包含一个偏移量和大小。 偏移量指向元素向量中数组的第一个元素。 Size 指定数组中元素的数量。

这是一个例子：

![../_images/array-vector.png](https://facebookincubator.github.io/velox/_images/array-vector.png)

我们将数组向量称为顶级向量，将元素向量称为嵌套或内部向量。 在上面的示例中，顶级数组有 4 个顶级行，元素数组有 11 个嵌套行。 前 3 个嵌套行对应于第 0 个顶级行。 接下来的 2 个嵌套行对应于第一个顶级行。 以下 4 个嵌套行对应于第 2 个顶级行。 其余 2 个嵌套行对应于第 3 个顶级行。

元素向量中的值可能不会以与数组向量中相同的顺序出现。 这是相同逻辑数组向量的替代布局示例。 在这里，元素数组中的值以不同的顺序出现，例如 首先是第 0 行的元素，然后是第 2 行的元素，然后是第 1 行的元素，然后是第 3 行的元素。 偏移量被调整为指向元素数组中的正确条目。 大小不变。

![../_images/array-out-of-order.png](https://facebookincubator.github.io/velox/_images/array-out-of-order.png)

同时具有偏移量和大小允许乱序写入数组向量，例如 在写条目 3 之前写条目 5。

通过将大小设置为零来指定空数组。 空数组的偏移量被认为是未定义的，可以是任何值。 考虑使用零作为空数组的偏移量。

空数组和空数组是不一样的。

元素向量可能具有独立于数组向量本身的空值缓冲区的空值缓冲区。 这允许我们指定具有部分或全部空元素的非空数组。 空数组和所有空元素的数组是不一样的。



### Maps

MapVector 存储类型为 MAP 的值。 除了空缓冲区，它还包含偏移量和大小缓冲区、键和值向量。 偏移量和大小是 32 位整数。

```
BufferPtr offsets_;
BufferPtr sizes_;
VectorPtr keys_;
VectorPtr values_;
```

这是一个例子：

![../_images/map-vector.png](https://facebookincubator.github.io/velox/_images/map-vector.png)

与数组向量类似，各个映射条目按顺序一起出现在键和值向量中。 然而，顶层映射#4 的映射条目不需要恰好出现在顶层映射#5 的映射条目之前。

通过将大小设置为零来指定空map。 空map的偏移量被认为是未定义的，可以是任何值。 考虑使用零作为空map的偏移量。

空映射和空映射是不一样的。

键和值向量可能具有彼此独立的空值缓冲区以及映射向量本身的空值缓冲区。 这允许我们指定一些或所有值为空的非空映射。 从技术上讲，map也可能有一个空键，尽管这在实践中可能不是很有用。 空映射和所有值为空的映射是不一样的。



### Structs

最后，RowVector 存储 ROW 类型的值（例如结构）。 除了空缓冲区之外，它还包含一个子向量列表。

```
std::vector<VectorPtr> children_;
```

![../_images/row-vector.png](https://facebookincubator.github.io/velox/_images/row-vector.png)

ROW 向量可以有任意数量的子向量，包括零。

对于 ROW 向量中的每一顶级行，每个子向量中都有一行。

子向量可能包含它们自己的空缓冲区，因此，可以有一个非空的顶级结构值，其中一些或所有子字段为空。 空结构与所有字段都为空的结构不同。 顶级结构为 null 的行的子字段的值未定义。

RowVector 用于表示 struct 类型的单个列以及在查询执行期间从一个运算符传递到下一个运算符的一组列。



## Constant Vector - Complex Types

复杂类型的常量向量由 ConstantVector<ComplexType> 表示，其中 ComplexType 是用于所有复杂类型的特殊标记类型。 ARRAY(INTEGER) 类型的常量向量和 MAP(TINYINT, VARCHAR) 类型的常量向量由同一类表示：ConstantVector<ComplexType>。

ConstantVector<ComplexType> 通过指向另一个向量中的特定行来标识特定的复杂类型值。

```
VectorPtr valueVector_;
vector_size_t index_;
```

下图显示了一个类型为 ARRAY(INTEGER) 的复数向量，表示一个由 3 个整数组成的数组：[10, 12, -1, 0]。 它被定义为指向其他一些 ArrayVector 中的第 2 行的指针。

![../_images/constant-array-vector.png](https://facebookincubator.github.io/velox/_images/constant-array-vector.png)

使用 BaseVector::wrapInConstant() 静态方法创建复杂类型的常量向量。 任何向量都可以包裹在一个常量向量中。 当您将 wrapInConstant() 与非平面向量一起使用时，生成的常量向量最终会引用最里面的向量，例如 wrapInConstant(100, 5, Dict (Flat)) 返回 ConstantVector<ComplexType>(100, Dict->wrappedIndex(5), Flat)。

```
static std::shared_ptr<BaseVector> wrapInConstant(
    vector_size_t length,
    vector_size_t index,
    std::shared_ptr<BaseVector> vector);
```

BaseVector 中定义的 WrappedVector() 虚拟方法提供了对底层平面向量的访问。

BaseVector 中定义的wrappedIndex(index) 虚拟方法将索引返回到标识常量值的底层平面向量中。 此方法为所有输入返回相同的值，因为常量向量的所有行都映射到底层平面向量的同一行。



## Dictionary Vector - Complex Type

与常量向量类似，复杂类型的字典向量由 DictionaryVector<ComplexType> 表示，其中 ComplexType 是用于所有复杂类型的特殊标记类型。 ARRAY (INTEGER) 类型的字典向量和 MAP(TINYINT, VARCHAR) 类型的字典向量由同一类表示：DictionaryVector<ComplexType>。 否则，复杂类型的字典向量与标量类型的字典向量没有什么不同。