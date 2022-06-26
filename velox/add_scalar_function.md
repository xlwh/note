# 如何开发一个标量函数？

## Simple Functions

本文档介绍 Velox 中简单函数 API 的主要概念、特性和示例。 如需更多真实的 API 使用示例, 请查看：**velox/example/SimpleFunctions.cpp**.

一个简单的标量函数，例如 一个数学函数，可以通过在模板类中包装一个 C++ 函数来添加。 例如，一个 ceil 函数可以实现为：

```
template <typename TExecParams>
struct CeilFunction {
  template <typename T>
  FOLLY_ALWAYS_INLINE void call(T& result, const T& a) {
    result = std::ceil(a);
  }
};
```

所有简单的函数类都需要模板化，并提供“调用”方法（或下面描述的变体之一）。 顶级模板参数提供类型系统适配器，它允许开发人员使用非原始类型，例如字符串、数组、映射和结构（请查看下面的示例）。 虽然顶层模板参数不用于对原始类型进行操作的函数，例如上面示例中的那个，但它仍然需要指定。

call 方法本身也可以被模板化或重载，以允许在不同的输入类型上调用函数，例如 浮动和双倍。 请注意，模板实例化只会在函数注册期间发生，如下面的“注册”部分所述。

请避免使用过时的 VELOX_UDF_BEGIN/VELOX_UDF_END 宏。

“Call”函数（或其变体之一）可能返回 (a) void 表示该函数从不返回空值，或 (b) 布尔值表示计算结果是否为空。 True 表示结果不为空； false 表示结果为空。 如果“ceil(0)”返回null，上面的函数可以重写如下：

```
template <typename TExecParams>
struct NullableCeilFunction {
  template <typename T>
  FOLLY_ALWAYS_INLINE bool call(T& result, const T& a) {
    result = std::ceil(a);
    return a != 0;
  }
};
```

参数列表必须以输出参数“result”开头，后跟函数参数。 “结果”参数必须是参考。 函数参数必须是 const 引用。 参数的 C++ 类型必须与以下映射中指定的 Velox 类型匹配：

| Velox Type | C++ Argument Type           | C++ Result Type             |
| :--------- | :-------------------------- | :-------------------------- |
| BOOLEAN    | bool                        | bool                        |
| TINYINT    | int8_t                      | int8_t                      |
| SMALLINT   | int16_t                     | int16_t                     |
| INTEGER    | int32_t                     | int32_t                     |
| BIGINT     | int64_t                     | int64_t                     |
| REAL       | float                       | float                       |
| DOUBLE     | double                      | double                      |
| VARCHAR    | StringView                  | out_type<Varchar>           |
| VARBINARY  | StringView                  | out_type<Varbinary>         |
| TIMESTAMP  | Timestamp                   | Timestamp                   |
| ARRAY      | arg_type<Array<E>>          | out_type<Array<E>>          |
| MAP        | arg_type<Map<K,V>>          | out_type<Map<K, V>>         |
| ROW        | arg_type<Row<T1, T2, T3,…>> | out_type<Row<T1, T2, T3,…>> |

arg_type 和 out_type 模板是使用类定义中的 VELOX_DEFINE_FUNCTION_TYPES(TExecParams) 宏定义的。 这些类型提供类似于 std::string、std::vector、std::unordered_map 和 std::tuple 的接口。 底层实现经过优化，可以在没有额外复制的情况下从列表示中读取和写入。

注意：暂时不要过多关注复杂类型映射。 为了完整起见，它们被包含在这里，但需要一个完整的单独讨论。

### Null Behavior

大多数函数都有默认的 null 行为，例如 任何参数中的空值都会产生空结果。 表达式评估引擎会自动为此类输入生成空值，省略对实际函数的调用。 如果给定函数对空输入有不同的行为，它必须定义一个“callNullable”函数而不是“call”函数。 下面是一个 ceil 函数的人工示例，它为 null 输入返回 0：

```
template <typename TExecParams>
struct CeilFunction {
  template <typename T>
  FOLLY_ALWAYS_INLINE void callNullable(T& result, const T* a) {
    // Return 0 if input is null.
    if (a) {
      result = std::ceil(*a);
    } else {
      result = 0;
    }
  }
};
```

请注意， callNullable 函数将参数作为原始指针而不是引用以允许指定空值。 callNullable() 还可以返回 void 以指示该函数不产生空值。

#### Null-Free Fast Path

“callNullFree”函数可以代替“call”和/或“callNullable”函数实现或与“call”和/或“callNullable”函数一起实现。 当仅实现“callNullFree”函数时，如果任何输入参数为空（如默认空行为）或任何输入参数为复杂类型，则将跳过该函数的评估并自动生成空值，并且 在其值的任何位置包含 null，例如 具有空元素的数组。 如果“callNullFree”与“call”和/或“callNullable”一起实现，则对批处理应用 O(N * D) 检查以查看任何输入参数是否可能为或包含 null，其中 N 是 输入参数，D 是复杂类型的嵌套深度。 只有当可以明确确定没有空值时才会调用“callNullFree”。 在这种情况下，“callNullFree”可以通过避免任何每行空检查来充当快速路径。

下面是一个 array_min 函数的示例，它返回数组中的最小值：

```
template <typename TExecParams>
struct ArrayMinFunction {
  VELOX_DEFINE_FUNCTION_TYPES(TExecParams);

  template <typename TInput>
  FOLLY_ALWAYS_INLINE bool callNullFree(
      TInput& out,
      const null_free_arg_type<Array<TInput>>& array) {
    out = INT32_MAX;
    for (auto i = 0; i < array.size(); i++) {
      if (array[i] < out) {
        out = array[i]
      }
    }
    return true;
  }
};
```

请注意，我们可以访问“array”的元素，而无需在“callNullFree”中检查它们的无效性。 另请注意，我们将输入类型包装在 null_free_arg_type<...> 模板中，而不是 arg_type<...> 模板中。 这是必需的，因为复杂类型的输入类型在“callNullFree”函数中属于不同类型，这些函数在访问时不会将值包装在类似 std::optional 的接口中。

### Determinism

默认情况下，假设简单函数是确定性的，例如 给定相同的输入，它们总是产生相同的结果。 如果不是这种情况，该函数必须定义一个静态 constexpr bool is_deterministic 成员：

```
static constexpr bool is_deterministic = false;
```

这种函数的一个例子是 rand()：

```
template <typename TExecParams>
struct RandFunction {
  static constexpr bool is_deterministic = false;

  FOLLY_ALWAYS_INLINE bool call(double& result) {
    result = folly::Random::randDouble01();
    return true;
  }
};
```

### All-ASCII Fast Path

处理字符串输入的函数必须对 UTF-8 输入正常工作。 但是，如果已知输入仅包含 ASCII 字符，则通常可以更有效地实现这些功能。 此类函数可以提供处理 UTF-8 字符串的“call”方法和处理纯 ASCII 字符串的“callAscii”方法。 引擎将检查输入字符串，如果输入全是 ASCII，则调用“callAscii”方法，如果输入可能包含多字节字符，则调用“call”方法。

此外，大多数接受字符串输入并产生字符串输出的函数都具有所谓的默认 ASCII 行为，例如 全 ASCII 输入保证全 ASCII 输出。 如果是这种情况，该函数可以通过定义 is_default_ascii_behavior 成员变量并将其初始化为 true 来指示。 引擎会自动将结果字符串标记为全 ASCII。 当这些字符串作为输入传递给其他函数时，引擎不需要扫描字符串来确定它们是否为 ASCII。

下面是一个Trim函数的例子：

```
template <typename TExecParams>
struct TrimFunction {
  VELOX_DEFINE_FUNCTION_TYPES(TExecParams);

  // ASCII input always produces ASCII result.
  static constexpr bool is_default_ascii_behavior = true;

  // Properly handles multi-byte characters.
  FOLLY_ALWAYS_INLINE bool call(
      out_type<Varchar>& result,
      const arg_type<Varchar>& input) {
    stringImpl::trimUnicodeWhiteSpace<leftTrim, rightTrim>(result, input);
    return true;
  }

  // Assumes input is all ASCII.
  FOLLY_ALWAYS_INLINE bool callAscii(
      out_type<Varchar>& result,
      const arg_type<Varchar>& input) {
    stringImpl::trimAsciiWhiteSpace<leftTrim, rightTrim>(result, input);
    return true;
  }
};
```

### Zero-copy String Result

substr() 和 trim() 等函数可以通过引用输入字符串来产生零拷贝结果。 为此，他们必须定义一个reuse_strings_from_arg 成员变量并将其初始化为参数的索引，该参数的字符串在结果中被重用。 这将允许引擎向结果向量添加对输入字符串缓冲区的引用，并确保这些缓冲区不会过早消失。 输出类型可以是标量字符串（varchar 和 varbinaries），也可以是包含字符串的复杂类型，例如数组、映射和行。

```
// Results refer to strings in the first argument.
static constexpr int32_t reuse_strings_from_arg = 0;
```

### Access to Session Properties and Constant Inputs

某些功能需要访问会话属性，例如会话的时区。 一些示例是 day()、hour() 和 minute() Presto 函数。 其他功能可以从预处理一些常量输入中受益，例如 编译正则表达式模式或解析日期和时间单位。 要访问会话属性和常量输入，该函数必须定义一个初始化方法，该方法接收对 QueryConfig 的常量引用和每个输入参数的常量指针列表。 常量输入将指定它们的值。 非常量的输入将作为 nullptr 传递。 initialize 方法的签名类似于 callNullable 方法的签名，但多了一个第一个参数 const core::QueryConfig&。 引擎在每个查询和执行线程中调用一次初始化方法。

这是从会话属性中提取时区并在处理输入时使用它的小时函数示例。

```
template <typename TExecParams>
struct HourFunction {
  VELOX_DEFINE_FUNCTION_TYPES(TExecParams);

  const date::time_zone* timeZone_ = nullptr;

  FOLLY_ALWAYS_INLINE void initialize(
      const core::QueryConfig& config,
      const arg_type<Timestamp>* /*timestamp*/) {
    timeZone_ = getTimeZoneFromConfig(config);
  }

  FOLLY_ALWAYS_INLINE bool call(
      int64_t& result,
      const arg_type<Timestamp>& timestamp) {
    int64_t seconds = getSeconds(timestamp, timeZone_);
    std::tm dateTime;
    gmtime_r((const time_t*)&seconds, &dateTime);
    result = dateTime.tm_hour;
    return true;
  }
};
```

这是 date_trunc() 函数在初始化期间解析常量单位参数并在处理单个行时重新使用解析值的另一个示例:

```
template <typename TExecParams>
struct DateTruncFunction {
  VELOX_DEFINE_FUNCTION_TYPES(TExecParams);

  const date::time_zone* timeZone_ = nullptr;
  std::optional<DateTimeUnit> unit_;

  FOLLY_ALWAYS_INLINE void initialize(
      const core::QueryConfig& config,
      const arg_type<Varchar>* unitString,
      const arg_type<Timestamp>* /*timestamp*/) {
    timeZone_ = getTimeZoneFromConfig(config);
    if (unitString != nullptr) {
      unit_ = fromDateTimeUnitString(*unitString);
    }
  }

  FOLLY_ALWAYS_INLINE bool call(
      out_type<Timestamp>& result,
      const arg_type<Varchar>& unitString,
      const arg_type<Timestamp>& timestamp) {
    const auto unit =
        unit_.has_value() ? unit_.value() : fromDateTimeUnitString(unitString);
    ...<use unit enum>...
  }
};
```

### Registration

使用 registerFunction 模板注册简单的函数:

```
template <template <class> typename Func, typename TReturn, typename... TArgs>
void registerFunction(
    const std::vector<std::string>& aliases = {},
    std::shared_ptr<const Type> returnType = nullptr)
```

第一个模板参数是类名，下一个模板参数是返回类型，其余模板参数是实参类型。 Aliases 参数允许开发者为同一个函数指定多个名称，但每个函数注册至少需要提供一个名称。 上面定义的“ceil”函数可以使用以下函数调用来注册：

```
registerFunction<CeilFunction, double, double>({"ceil", "ceiling");
```

在这里，我们注册 CeilFunction 函数，它接受一个双精度并返回一个双精度。 如果我们想允许在浮点输入上调用 ceil 函数，我们需要再次调用 registerFunction：

```
registerFunction<CeilFunction, float, float>({"ceil", "ceiling");
```

我们需要为我们想要支持的每个签名调用 registerFunction。

### Codegen

要允许在代码生成中使用该函数，请将函数的“内核”提取到一个头文件中，然后从“call”或“callNullable”中调用它。 这是一个带有 ceil 函数的示例。

```
#include "velox/functions/prestosql/ArithmeticImpl.h"

template <typename TExecParams>
struct CeilFunction {
  template <typename T>
  FOLLY_ALWAYS_INLINE bool call(T& result, const T& a) {
    result = ceil(a);
    return true;
  }
};
```

velox/functions/prestosql/ArithmeticImpl.h:

```
template <typename T>
T ceil(const T& arg) {
  T results = std::ceil(arg);
  return results;
}
```

确保定义“kernels”的头文件尽可能地没有依赖关系，以便在 codegen 中更快地编译。

### Complex Types

#### Inputs (View Types)

输入复杂类型在简单函数接口中使用轻量级惰性访问抽象来表示，从而可以有效地直接访问 Velox 向量中的底层数据。 如前所述，辅助别名 arg_type 和 null_free_arg_type 可用于函数签名中，以将 Velox 类型映射到相应的输入类型。 下表显示了用于表示不同复杂类型的输入的实际类型。

| C++ Argument Type            | C++ Actual Argument Type | Corresponding std type        |
| :--------------------------- | :----------------------- | :---------------------------- |
| arg_type<Array<E>>           | NullableArrayView<E>>    | std::vector<std::optional<V>> |
| arg_type<Map<K,V>>           | NullableMapView<K, V>    | std::map<K, std::optional<V>> |
| arg_type<Row<T…>>            | NullableRowView<T…>      | std::tuple<std::optional<T>…  |
| null_free_arg_type<Array<E>> | NullFreeArrayView<E>     | std::vector<V>                |
| null_free_arg_type<Map<K,V>> | NullFreeMapView<K, V>    | std::map<K, V>                |
| null_free_arg_type<Row<T…>>> | NullFreeRowView<T…>      | std::tuple<T…>                |

视图类型被设计为具有类似于 std::containers 的接口，事实上在大多数情况下它们可以用作替代品。 上表显示了 Velox 类型和对应的 std 类型之间的映射。 例如：一个 Map<Row<int, int>, Array<float>> 对应于 const std::map<std:::tuple<int, int>, std::vector<float>>。

所有视图类型复制对象都很轻量，例如 ArrayView 的大小最大为 16 字节。

**OptionalAccessor<E>**:

OptionalAccessor 是一个类似 std::optional 的对象，它提供对特定索引处底层 Velox 向量的空值和值的惰性访问。 目前，它用于表示可空输入数组的元素和可空输入映射的值。 请注意，映射中的键在 Velox 中被假定为始终不可为空

该对象支持以下方法：

- arg_type<E> value() : unchecked access to the underlying value.
- arg_type<E> operator [*](https://facebookincubator.github.io/velox/develop/scalar-functions.html#id1)() : unchecked access to the underlying value.
- bool has_value() : return true if the value is not null.
- bool operator() : return true if the value is not null.

nullity 和 value 访问是分离的，因此如果有人知道输入是 null-free 的，那么访问 value 就不会产生检查 nullity 的开销。 检查无效性也是如此。 请注意，与 std::container 不同，对 value() 和 operator* 的函数调用是右值（临时）而不是左值，它们可以绑定到 const 引用和左值但不能绑定到引用。

对于原始类型，OptionalAccessor<E> 可分配给 std::optional<arg_type<E>> 并与之比较。 以下表达式是有效的，其中 array[0] 是可选的访问器。

```
std::optional<int> = array[0];
if(array[0] == std::nullopt) ...
if(std::nullopt == array[0]) ...
if(array[0]== std::optional<int>{1}) ...
```

**NullableArrayView<T> and NullFreeArrayView<T>**

NullableArrayView 和 NullFreeArrayView 具有类似于 std::vector<std::optional<V>> 和 std::vector<V> 的接口，下面的代码显示了函数 arraySum，一个范围循环用于迭代值。

```
template <typename T>
struct ArraySum {
  VELOX_DEFINE_FUNCTION_TYPES(T);

  bool call(const int64_t& output, const arg_type<Array<int64_t>>& array) {
    output = 0;
    for(const auto& element : array) {
      if (element.has_value()) {
        output += element.value();
      }
    }
    return true;
  }
};
```

ArrayView 支持以下内容：

- size_t size() : return the number of elements in the array.
- operator[](size_t index) : access element at index. It returns either null_free_arg_type<T> or OptionalAccessor<T>.
- ArrayView<T>::Iterator begin() : iterator to the first element.
- ArrayView<T>::Iterator end() : iterator indicating end of iteration.
- bool mayHaveNulls() : constant time check on the underlying vector nullity. When it returns false, there are definitely no nulls, a true does not guarantee null existence.
- ArrayView<T>::SkipNullsContainer SkipNulls() : return an iterable container that provides direct access to non-null values in the underlying array. For example, the function above can be written as:

```
template <typename T>
struct ArraySum {
  VELOX_DEFINE_FUNCTION_TYPES(T);

  bool call(const int64_t& output, const arg_type<Array<int64_t>>& array) {
    output = 0;
    for (const auto& value : array.skipNulls()) {
      output += value;
    }
    return true;
  }
};
```

skipNulls 迭代器将检查每个索引处的空值并跳过空值，当 mayHaveNulls() 为假时，性能更高的实现将跳过读取空值。

```
template <typename T>
struct ArraySum {
    VELOX_DEFINE_FUNCTION_TYPES(T);

    bool call(const int64_t& output, const arg_type<Array<int64_t>>& array) {
      output = 0;
      if (array.mayHaveNulls()) {
        for(const auto& value : array.skipNulls()) {
          output += value;
        }
        return true;
      }

      // No nulls, skip reading nullity.
      for (const auto& element : array) {
        output += element.value();
      }
      return true;
    }
};
```

注意：对 operator[]、迭代器取消引用和迭代器指针取消引用的调用是 r -values（临时），而不是 STD 容器中的l-values。 因此，这些可以绑定到 const 引用或l-values，但不能绑定到普通引用。

**NullableMapView<K, V> and NullFreeMapView<K, V>**

NullableMapView 和 NullFreeMapView 具有类似于 std::map<K, std::optional<V>> 和 std::map<K, V> 的接口，下面的代码显示了一个示例函数 mapSum，对键和值求和。

```
template <typename T>
struct MapSum{
  bool call(const int64_t& output, const arg_type<Map<int64_t, int64_t>>& map) {
    output = 0;
    for (const auto& [key, value] : map) {
      output += key;
      if (value.has_value()) {
        value += value.value();
      }
    }
    return true;
  }
};
```

MapView 支持以下内容：

- MapView<K,V>::Element begin() : iterator to the first map element.
- MapView<K,V>::Element end() : iterator that indicates end of iteration.
- size_t size() : number of elements in the map.
- MapView<K,V>::Iterator find(const key_t& key): performs a linear search for the key, and returns iterator to the element if found otherwise returns end(). Only supported for primitive key types.
- MapView<K,V>::Iterator operator[](const key_t& key): same as find, throws an exception if element not found.
- MapView<K,V>::Element

MapView<K, V>::Element is the type returned by dereferencing MapView<K, V>::Iterator. It has two members:

- first : arg_type<K> | null_free_arg_type<K>
- second: OptionalAccessor<V> | null_free_arg_type<V>
- MapView<K, V>::Element participates in struct binding: auto [v, k] = [*](https://facebookincubator.github.io/velox/develop/scalar-functions.html#id3)map.begin();

**Temporaries lifetime C++**

虽然 c++ 允许临时变量（r-values）通过延长它们的生命周期来绑定到 const 引用，但必须小心并知道只有分配的临时生命周期被延长，而不是 RHS 表达式链中的所有临时变量。 换句话说，表达式中任何临时的生命周期都不会延长

例如，对于表达式 const auto& x = map.begin()->first。 c++ 不会延长 map.begin() 结果的生命周期，因为它不是被分配的。 在这种情况下，赋值具有未定义的行为。

```
// Safe assignments. single rhs temporary.
const auto& a = array[0];
const auto& b = *a;
const auto& c = map.begin();
const auto& d = c->first;

// Unsafe assignments. (undefined behaviours)
const auto& a = map.begin()->first;
const auto& b = **it;

// Safe and cheap to assign to value.
const auto a = map.begin()->first;
const auto b = **it;
```

请注意，在范围循环中，范围表达式被分配给一个通用引用。 因此，上述问题适用于它。

```
// Unsafe range loop.
for(const auto& e : **it){..}

// Safe range loop.
auto itt = *it;
for(const auto& e : *itt){..}
```



#### Outputs (Writer Types)

复杂类型的输出使用特殊编写器表示，这些编写器的设计方式通过直接写入 Velox 向量来最大限度地减少数据复制。

**ArrayWriter<V>**

- out_type<V>& add_item() : add non-null item and return the writer of the added value.
- add_null(): add null item.
- reserve(vector_size_t size): make sure space for size items is allocated in the underlying vector.
- vector_size_t size(): get the length of the array.
- resize(vector_size_t size): change the size of the array reserving space for the new elements if needed.
- void add_items(const T& data): append data from any container with std::vector-like interface.
- void copy_from(const T& data): assign data to match that of any container with std::vector-like interface.
- void add_items(const NullFreeArrayView<V>& data): append data from array view (faster than item by item).
- void copy_from(const NullFreeArrayView<V>& data): assign data from array view (faster than item by item).
- void add_items(const NullableArrayView<V>& data): append data from array view (faster than item by item).
- void copy_from(const NullableArrayView<V>& data): assign data from array view (faster than item by item).

当 V 是原始的时，以下函数可用，使编写器可用作 std::vector<V>。

- push_back(std::optional<V>): add item or null.
- PrimitiveWriter<V> operator[](vector_size_t index): return a primitive writer that is assignable to std::optional<V> for the item at index (should be called after a resize).
- PrimitiveWriter<V> back(): return a primitive writer that is assignable to std::optional<V> for the item at index length -1.

**MapWriter<K, V>**

- reserve(vector_size_t size): make sure space for size entries is allocated in the underlying vector.
- std::tuple<out_type<K>&, out_type<V>&> add_item(): add non-null item and return the writers of key and value as tuple.
- out_type<K>& add_null(): add null item and return the key writer.
- vector_size_t size(): return the length of the array.
- void add_items(const T& data): append data from any container with std::vector<tuple<K, V>> like interface.
- void copy_from(const NullFreeMapView<V>& data): assign data from array view (faster than item by item).
- void copy_from(const NullableMapView<V>& data): assign data from array view (faster than item by item).

When K and V are primitives, the following functions are available, making the writer usable as std::vector<std::tuple<K, V>>.

- resize(vector_size_t size): change the size.
- emplace(K, std::optional<V>): add element to the map.
- std::tuple<K&, PrimitiveWriter<V>> operator[](vector_size_t index): returns pair of writers for element at index. Key writer is assignable to K. while value writer is assignable to std::optional<V>.

**RowWriter<T…>**

- template<vector_size_t I> set_null_at(): set null for row item at index I.
- template<vector_size_t I> get_writer_at(): set not null for row item at index I, and return writer to to the row element at index I.

When all types T… are primitives, the following functions are available.

- void operator=(const std::tuple<T…>& inputs): assignable to std::tuple<T…>.
- void operator=(const std::tuple<std::optional<T>…>& inputs): assignable to std::tuple<std::optional<T>…>.
- void copy_from(const std::tuple<K…>& inputs): similar as the above.

When a given Ti is primitive, the following is valid.

- PrimitiveWriter<Ti> exec::get<I>(RowWriter<T…>): return a primitive writer for item at index I that is assignable to std::optional.

**PrimitiveWriter<T>**

可赋值给 std::optional<T> 允许将 null 或值写入原语。 在编写可为空的原语时由复杂的编写者返回。

**StringWriter<>**:

- void reserve(size_t newCapacity) : Reserve a space for the output string with size of at least newCapacity.
- void resize(size_t newCapacity) : Set the size of the string.
- char* data(): returns pointer to the first char of the string, can be written to directly (safe to write to index at capacity()-1).
- vector_size_t capacity(): returns the capacity of the string.
- vector_size_t size(): returns the size of the string.
- operator+=(const T& input): append data from char* or any type with data() and size().
- append(const T& input): append data from char* or any type with data() and size().
- copy_from(const T& input): append data from char* or any type with data() and size().

当启用零拷贝优化时（参见上面的零拷贝字符串结果部分），可以使用以下函数。

- void setEmpty(): set to empty string.
- void setNoCopy(const StringView& value): set string to an input string without performing deep copy.

#### Limitations

1. 无法定义具有通用输出类型的函数，例如 Array<T> 输出，其中 T 是输入类型。
2. 如果函数在写入复杂类型时抛出异常，则正在写入的行的输出以及下一行的输出是未定义的。 因此，对于函数内的复杂输出，建议在开始写入后避免引发异常

### Variadic Arguments

简单函数的最后一个参数可以标记为“可变参数”。 这意味着此函数的调用可能在调用结束时包含该类型的 0..N 个参数。 虽然在 Velox 中不是真正的类型，但“Variadic”可以被认为是一种句法类型，其行为与 Array 有点相似。

| C++ Argument Type               | C++ Actual Argument Type |
| :------------------------------ | :----------------------- |
| arg_type<Variadic<E>>           | NullableVariadicView<E>  |
| null_free_arg_type<Variadic<E>> | NullFreeVariadicView<E>  |

与 NullableArrayView 和 NullFreeArrayView 一样，VariadicViews 具有与 const std::vector<std::optional<V>> 类似的接口。

NullableVariadicView 和 NullFreeVariadicView 支持以下内容：

- size_t size() : return the number of arguments that were passed as part of the “Variadic” type in the function invocation.
- operator[](size_t index) : access the value of the argument at index. It returns either null_free_arg_type<E> or OptionalAccessor<E>.
- VariadicView<T>::Iterator begin() : iterator to the first argument.
- VariadicView<T>::Iterator end() : iterator indicating end of iteration.
- bool mayHaveNulls() : a check on the nullity of the arugments (note this takes time proportional to the number of arguments). When it returns false, there are definitely no nulls, a true does not guarantee null existence.
- VariadicView<T>::SkipNullsContainer SkipNulls() : return an iterable container that provides direct access to each argument with a non-null value.

下面的代码显示了一个连接可变数量字符串的函数示例：

```
template <typename T>
struct VariadicArgsReaderFunction {
  VELOX_DEFINE_FUNCTION_TYPES(T);

  FOLLY_ALWAYS_INLINE bool call(
      out_type<Varchar>& out,
      const arg_type<Variadic<Varchar>>& inputs) {
    for (const auto& input : inputs) {
      if (input.has_value()) {
        output += input.value();
      }
    }

    return true;
  }
};
```

## Vector Functions

简单函数处理单行并生成单个值作为结果。 向量函数处理一批或多行并产生结果向量。 这些功能的一些定义特征是：

- 将向量作为输入并生成向量作为结果
- 可以访问矢量编码和元数据；
- 可以为通用输入类型定义，例如 通用数组、映射和结构；
- 允许实现 lambda 函数；

矢量函数接口允许许多简单函数不可用的优化。 这些优化通常利用不同的向量编码和向量的列表示。 这里有些例子:

- [`map_keys()`](https://facebookincubator.github.io/velox/functions/map.html#map_keys)  函数利用 ArrayVector 表示并简单地返回内部“键”向量而不进行任何计算。 类似地，map_values() 函数只返回内部“值”向量。
- [`map_entries()`](https://facebookincubator.github.io/velox/functions/map.html#map_entries) 函数获取输入向量的各个部分——“nulls”、“sizes”和“offsets”缓冲区以及“keys”和“values”向量——并以 RowVector 的形式简单地重新打包它们。
- [`cardinality()`](https://facebookincubator.github.io/velox/functions/array.html#cardinality) 函数利用 ArrayVector 和 MapVector 表示，并简单地返回输入向量的“大小”缓冲区。
- [`is_null()`](https://facebookincubator.github.io/velox/functions/comparison.html#is_null) 函数复制输入向量的“nulls”缓冲区，批量翻转位并返回结果。
- [`element_at()`](https://facebookincubator.github.io/velox/functions/array.html#element_at)  数组和映射的函数和下标运算符使用字典编码来表示输入“元素”或“值”向量的子集，而无需复制

要定义向量函数，请创建 exec::VectorFunction 的子类并实现“apply”方法。

```
void apply(
      const SelectivityVector& rows,
      std::vector<VectorPtr>& args,
      Expr* caller,
      EvalCtx* context,
      VectorPtr* result) const
```

### Input rows

“rows”参数指定传入批处理中要处理的行集。 此集合可能不包括所有行。 默认情况下，假定向量函数具有默认的空行为，例如 任何输入中的 null 都会产生 null 结果。 在这种情况下，表达式评估引擎将从“应用”调用中指定的“行”中排除具有空值的行。 如果函数对 null 输入具有不同的行为，则它必须重写 isDefaultNullBehavior 方法以返回 false。

```
bool isDefaultNullBehavior() const override {
  return false;
}
```

在这种情况下，“rows”参数将包括具有空输入的行，并且函数将需要处理这些。 默认情况下，该函数可以假设所有“行”的所有输入都不为空。

当执行一个函数作为条件表达式的一部分时，例如 AND、OR、IF、SWITCH，“行”的集合表示需要评估的行的子集。 考虑一些例子。

```
a > 5 AND b > 7
```

在这里，a > 5 在“a”不为空的所有行上计算，但 b > 7 在 b 不为空且 a 为空或不 > 5 的行上计算。

```
IF(condition, a + 5, b - 3)
```

在这里，a + 5 在 a 不为空且条件为真的行上计算，而 b - 3 在 b 不为空且条件不为真的行上计算。

在某些情况下，“行”之外的值可能未定义、未初始化或包含垃圾。 如果较早的过滤器操作生成字典编码的向量，其索引指向通过过滤器的行的子集，就会出现这种情况。 在计算 f(g(a)) 时，其中 a = Dict (a0)，函数“g”在“a0”中的行子集上进行计算，并且可能产生仅填充该行子集的结果。 然后，函数“f”在“g”结果中的相同行子集上进行评估。 “f”的输入将具有“行”之外的未定义、未初始化或包含垃圾的值。

请注意，SelectivityVector::applyToSelected 方法可用于以类似于标准 for 循环的方式循环指定行。

```
rows.applyToSelected([&] (auto row) {
    // row is the 0-based row number
    // .... process the row
});
```

### Input vectors

“args”参数是包含函数参数值的 Velox 向量的 std::vector。 这些向量不一定是平面的，可能是字典或常量编码的。 但是，采用单个参数的确定性函数保证接收其唯一输入作为平面向量。 默认情况下，假定函数是确定性的。 如果不是这种情况，该函数必须重写 isDeterministic 方法以返回 false。

```
bool isDeterministic() const override {
  return false;
}
```

请注意，DecodedVector 可用于获取到任何向量的平面向量接口。 辅助类 exec::DecodedArgs 可用于解码多个参数。

```
exec::DecodedArgs decodedArgs(rows, args, context);

auto firstArg = decodedArgs.at(0);
auto secondArg = decodedArgs.at(1);
```

### Result vector

“result”参数是一个指向 VectorPtr 的原始指针，它是一个指向 BaseVector 的 std::shared_ptr。 它可以为空，可以指向一个可重用的临时向量或必须保留其内容的部分填充向量。

在评估 IF 的“else”分支时，指定了部分填充的向量。 在这种情况下，必须保留“then”分支的结果。 这可以通过遵循两种模式之一轻松实现。

将所有或仅指定行的结果计算到一个新向量中，然后使用 EvalCtx::moveOrCopyResult 方法将向量 std::move 到“结果”或将单个行复制到部分填充的“结果”中。

下面是一个使用 moveOrCopyResult 实现 map_keys 函数的例子：

```
void apply(
    const SelectivityVector& rows,
    std::vector<VectorPtr>& args,
    exec::Expr* /* caller */,
    exec::EvalCtx* context,
    VectorPtr* result) const override {
  auto mapVector = args[0]->as<MapVector>();
  auto mapKeys = mapVector->mapKeys();

  auto localResult = std::make_shared<ArrayVector>(
      context->pool(),
      ARRAY(mapKeys->type()),
      mapVector->nulls(),
      rows.end(),
      mapVector->offsets(),
      mapVector->sizes(),
      mapKeys,
      mapVector->getNullCount());

  context->moveOrCopyResult(localResult, rows, result);
}
```

使用 BaseVector::ensureWritable 方法将“result”初始化为一个平面唯一引用的向量，同时保留“rows”中未指定的行中的值。 然后，计算并填写“结果”中的“行”。 如果“result”为空，BaseVector::ensureWritable 创建一个新向量。 如果结果不为空，但不平坦或不单独引用，则 BaseVector::ensureWritable 创建一个新向量并将“结果”中的非“行”值复制到新创建的向量中。 如果“result”不为空且平坦，BaseVector::ensureWritable 检查内部缓冲区，如果它们没有被单独引用，则复制这些缓冲区。 BaseVector::ensureWritable 还在内部向量（数组的元素向量、map 的键和值、结构的字段）上递归调用自身，以确保该向量自始至终都是“可写的”。

下面是使用 BaseVector::ensureWritable 为地图实现基数函数的示例：

```
void apply(
    const SelectivityVector& rows,
    std::vector<VectorPtr>& args,
    exec::Expr* /* caller */,
    exec::EvalCtx* context,
    VectorPtr* result) const override {

  BaseVector::ensureWritable(rows, BIGINT(), context->pool(), result);
  BufferPtr resultValues =
      (*result)->as<FlatVector<int64_t>>()->mutableValues(rows.size());
  auto rawResult = resultValues->asMutable<int64_t>();

  auto mapVector = args[0]->as<MapVector>();
  auto rawSizes = mapVector->rawSizes();

  rows.applyToSelected([&](vector_size_t row) {
    rawResult[row] = rawSizes[row];
  });
}
```

### Simple implementation

矢量函数接口非常灵活，可以进行许多有趣的优化。 也可能感觉很复杂。 让我们看看我们如何使用 DecodedVector 和 BaseVector::ensureWritable 将“power(a, b)”函数实现为向量函数，其方式并不比简单函数复杂多少。 澄清一下，最好将“power”函数实现为一个简单的函数。 我在这里仅出于说明目的使用它。

```
// Initialize flat results vector.
BaseVector::ensureWritable(rows, DOUBLE(), context->pool(), result);
auto rawResults = (*result)->as<FlatVector<int64_t>>()->mutableRawValues();

// Decode the arguments.
DecodedArgs decodedArgs(rows, args, context);
auto base = decodedArgs.decodedVector(0);
auto exp = decodedArgs.decodedVector(1);

// Loop over rows and calculate the results.
rows.applyToSelected([&](int row) {
  rawResults[row] =
      std::pow(base->valueAt<double>(row), exp->valueAt<double>(row));
});
```

您可能希望针对底数和指数均平坦的情况进行优化，并消除调用 DecodedVector::valueAt 模板的开销。

```
if (base->isIdentityMapping() && exp->isIdentityMapping()) {
  auto baseValues = base->values<double>();
  auto expValues = exp->values<double>();
  rows.applyToSelected([&](int row) {
    rawResults[row] = std::pow(baseValues[row], expValues[row]);
  });
} else {
  rows.applyToSelected([&](int row) {
    rawResults[row] =
        std::pow(base->valueAt<double>(row), exp->valueAt<double>(row));
  });
}
```

您可以决定进一步优化平底和常数指数的情况。

```
if (base->isIdentityMapping() && exp->isIdentityMapping()) {
  auto baseValues = base->values<double>();
  auto expValues = exp->values<double>();
  rows.applyToSelected([&](int row) {
    rawResults[row] = std::pow(baseValues[row], expValues[row]);
  });
} else if (base->isIdentityMapping() && exp->isConstantMapping()) {
  auto baseValues = base->values<double>();
  auto expValue = exp->valueAt<double>(0);
  rows.applyToSelected([&](int row) {
    rawResults[row] = std::pow(baseValues[row], expValue);
  });
} else {
  rows.applyToSelected([&](int row) {
    rawResults[row] =
        std::pow(base->valueAt<double>(row), exp->valueAt<double>(row));
  });
}
```

希望您现在可以看到实现中的额外复杂性仅来自于引入优化路径。 开发人员需要根据具体情况决定这种复杂性是否合理。

### TRY expression suppor

内置的 TRY 表达式计算输入表达式并通过返回 NULL 来处理某些类型的错误。 它用于在遇到损坏或无效数据时查询生成 NULL 或默认值而不是失败的情况。 要指定默认值，可以将 TRY 表达式与 COALESCE 函数结合使用

TRY 表达式的实现依赖于 VectorFunction 实现来调用 EvalCtx::setError(row, exception) 而不是直接抛出异常。

```
void setError(vector_size_t index, const std::exception_ptr& exceptionPtr);
```

一个典型的模式是遍历行，应用包装在 try-catch 中的函数并调用 context->setError(row, std::current_exception()); 从捕获块。

```
rows.applyToSelected([&](auto row) {
  try {
    // ... calculate and store the result for the row
  } catch (const std::exception& e) {
    context->setError(row, std::current_exception());
  }
});
```

有一个 EvalCtx::applyToSelectedNoThrow 便捷方法可以用来代替上面的显式 try-catch 块：

```
context->applyToSelectedNoThrow(rows, [&](auto row) {
  // ... calculate and store the result for the row
});
```

默认情况下，简单函数与 TRY 表达式兼容。 该框架将“call”和“callNullable”方法包装在try-catch 中，并使用context->setError 报告错误。

### Registration

使用 exec::registerVectorFunction 注册一个无状态向量函数。

```
bool registerVectorFunction(
    const std::string& name,
    std::vector<FunctionSignaturePtr> signatures,
    std::unique_ptr<VectorFunction> func,
    bool overwrite = true)
```

exec::registerVectorFunction 将名称、支持的签名列表和 unique_ptr 传递给函数的实例。 如果具有指定名称的函数已经存在，可选的“覆盖”标志指定是否覆盖函数。

使用 exec::registerStatefulVectorFunction 注册一个有状态向量函数。

```
bool registerStatefulVectorFunction(
    const std::string& name,
    std::vector<FunctionSignaturePtr> signatures,
    VectorFunctionFactory factory,
    bool overwrite = true)
```

exec::registerStatefulVectorFunction 接受名称、支持的签名列表和可用于创建向量函数实例的工厂函数。 表达式评估引擎使用工厂函数为每个执行线程创建向量函数的新实例。 在单线程执行中，函数的单个实例用于处理所有批次的数据。 在多线程执行中，每个线程创建一个单独的函数实例。

使用函数名称、类型和可选的参数常量值调用工厂函数。 例如，正则表达式函数通常用常量正则表达式调用。 有状态向量函数可以编译一次正则表达式（每个执行线程），并将编译后的表达式重新用于多批数据。 类似地，与常量 IN 列表一起使用的 IN 表达式可以创建一次值的哈希集，并将其重用于所有批次的数据。

```
// Represents arguments for stateful vector functions. Stores element type, and
// the constant value (if supplied).
struct VectorFunctionArg {
  const TypePtr type;
  const VectorPtr constantValue;
};

using VectorFunctionFactory = std::function<std::shared_ptr<VectorFunction>(
    const std::string& name,
    const std::vector<VectorFunctionArg>& inputArgs)>;
```

### Function signature

推荐使用 FunctionSignatureBuilder 创建 FunctionSignature 实例。 FunctionSignatureBuilder 和 FunctionSignature 支持类似 Java 的泛型、可变数量的参数和 lambda。 这里有些例子。

length 函数接受一个 varchar 类型的参数并返回一个 bigint：

```
// varchar -> bigint
exec::FunctionSignatureBuilder()
  .returnType("bigint")
  .argumentType("varchar")
  .build()
```

substr 函数接受一个 varchar 和两个整数作为开始和长度。 要指定多个参数的类型，请按顺序为每个参数调用 argumentType() 方法。

```
// varchar, integer, integer -> bigint
exec::FunctionSignatureBuilder()
  .returnType("varchar")
  .argumentType("varchar")
  .argumentType("integer")
  .argumentType("integer")
  .build()
```

concat 函数接受任意数量的 varchar 输入并返回一个 varchar。 FunctionSignatureBuilder 允许通过调用 variableArity() 方法指定最后一次扩充可能出现零次或多次。

```
// varchar... -> varchar
exec::FunctionSignatureBuilder()
    .returnType("varchar")
    .argumentType("varchar")
    .variableArity()
    .build()
```

map_keys 函数接受任何地图并返回地图键数组。

```
// map(K,V) -> array(K)
exec::FunctionSignatureBuilder()
  .typeVariable("K")
  .typeVariable("V")
  .returnType("array(K)")
  .argumentType("map(K,V)")
  .build()
```

transform 函数接受一个数组和一个 lambda，将 lambda 应用于数组的每个元素并返回一个新的结果数组。

```
// array(T), function(T, U) -> array(U)
exec::FunctionSignatureBuilder()
  .typeVariable("T")
  .typeVariable("U")
  .returnType("array(U)")
  .argumentType("array(T)")
  .argumentType("function(T, U)")
  .build();
```

FunctionSignatureBuilder 中使用的类型名称可以是小写标准类型、特殊类型“any”，也可以是调用 typeVariable() 方法定义的类型。 “any”类型可用于指定类似 printf 的函数，该函数接受任意数量的任何可能不匹配类型的参数。

## Testing

使用 velox/functions/prestosql/tests/FunctionBaseTest.h 中的 FunctionBaseTest 作为基类添加测试。 将您的测试和 .cpp 文件命名为 <function-name>Test，例如 CardinalityTest.cpp 中的 CardinalityTest 或 IsNullTest.cpp 中的 IsNullTest。

FunctionBaseTest 有许多用于生成测试向量的辅助方法。 它还提供了一个 evaluate() 方法，该方法接受 SQL 表达式和输入数据，计算表达式并返回结果向量。 SQL 表达式使用 DuckDB 解析，类型解析逻辑利用注册期间指定的函数签名。 assertEqualVectors() 方法采用两个向量，预期的和实际的，并断言它们表示相同的值。 向量的编码可能不相同。

```
TEST_F(ArrayContainsTest, integerWithNulls) {
  auto arrayVector = makeNullableArrayVector<int64_t>(
      {{1, 2, 3, 4},
       {3, 4, 5},
       {},
       {5, 6, std::nullopt, 7, 8, 9},
       {7, std::nullopt},
       {10, 9, 8, 7}});

  auto testContains = [&](std::optional<int64_t> search,
                          const std::vector<std::optional<bool>>& expected) {
    auto result = evaluate<SimpleVector<bool>>(
        "contains(c0, c1)",
        makeRowVector({
            arrayVector,
            makeConstant(search, arrayVector->size()),
        }));

    assertEqualVectors(makeNullableFlatVector<bool>(expected), result);
  };

  testContains(1, {true, false, false, std::nullopt, std::nullopt, false});
  testContains(3, {true, true, false, std::nullopt, std::nullopt, false});
  testContains(5, {false, true, false, true, std::nullopt, false});
  testContains(7, {false, false, false, true, true, true});
  testContains(-2, {false, false, false, std::nullopt, std::nullopt, false});
}
```

简单函数的测试可以受益于使用 evaluateOnce () 模板，该模板采用 SQL 表达式和标量值作为输入，在长度为 1 的向量上计算表达式并返回标量结果。 下面是一个简单函数“sqrt”的测试示例：

```
TEST_F(ArithmeticTest, sqrt) {
  constexpr double kDoubleMax = std::numeric_limits<double>::max();
  const double kNan = std::numeric_limits<double>::quiet_NaN();

  const auto sqrt = [&](std::optional<double> a) {
    return evaluateOnce<double>("sqrt(c0)", a);
  };

  EXPECT_EQ(1.0, sqrt(1));
  EXPECT_TRUE(std::isnan(sqrt(-1.0).value_or(-1)));
  EXPECT_EQ(0, sqrt(0));

  EXPECT_EQ(2, sqrt(4));
  EXPECT_EQ(3, sqrt(9));
  EXPECT_FLOAT_EQ(1.34078e+154, sqrt(kDoubleMax).value_or(-1));
  EXPECT_EQ(std::nullopt, sqrt(std::nullopt));
  EXPECT_TRUE(std::isnan(sqrt(kNan).value_or(-1)));
}
```

## Function names

对于简单函数和向量函数，它们的名称不区分大小写。 当函数被注册并为给定表达式解析时，函数名称会自动转换为小写。

## Benchmarking

使用 velox/functions/lib/benchmarks/FunctionBenchmarkBase.h 中的 folly::Benchmark 框架和 FunctionBenchmarkBase 作为基类添加基准测试。 基准是检查优化是否有效、评估它带来多少好处并决定是否值得增加复杂性的好方法。



## Documenting

如果一个函数实现了 Presto 语义，则通过向 velox/docs/functions 中的 *.rst 文件之一添加一个条目来记录它。 每个文件记录一组相关的功能。 例如。 math.rst 包含所有数学函数，而 array.rst 文件包含所有数组函数。 在文件中，函数按字母顺序列出。