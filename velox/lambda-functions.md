# 如何增加一个lambda函数？

## Introduction

Velox 支持用于在数组和映射上实现计算的 lambda 函数。 Lambda 函数是高阶函数，其参数本身就是函数。 例如，filter() 函数接受一个数组或映射和一个谓词，并返回传递谓词的数组或映射元素的子集。 transform() 函数接受一个数组和一个函数，将该函数应用于数组的每个元素并返回一个结果数组。

这是在 Presto SQL 中使用过滤器和转换函数的示例：

```
> select filter(array[1, 2, 3, 4], x -> x % 2 = 0);
 [2, 4]

> select transform(array[1, 2, 3, 4], x -> x * 2)
 [2, 4, 6, 8]
```

“x -> x % 2 = 0”是一个 lambda 表达式。 它由一个签名和一个由“箭头”分隔的正文组成。 请注意，在 Presto SQL 中，lambda 的签名仅包含参数名称。 参数类型是从上下文中推断出来的，例如 来自“filter”的数组参数的类型。

Lambda 可以使用捕获来访问封闭函数范围内的任何列。 假设我们有一个包含数组列“a”和整数列“b”的数据集：

| a: array(integer) | b: integer |
| :---------------- | :--------- |
| [1, 2, 3, 4]      | 3          |
| [3, 1, 5, 6, 7]   | 4          |

我们可以过滤数组“a”以仅保留大于或等于“b”的元素：

```
> select filter(a, x -> x >= b)

[3, 4]
[5, 6, 7]
```

这里，lambda 表达式中的“b”是捕获。 Lambda 表达式可以使用零个或多个捕获。

此外，可以将不同的 lambda 表达式应用于数据集中的不同行。 例如，我们可以过滤数组“a”以在“b”为偶数时保留偶数元素，如果“b”为奇数则保留奇数元素。

```
> select filter(a, if(b % 2 == 0, x -> x % 2 == 0, x % 2 == 1))

[1, 3]
[6]
```

注意：在撰写本文时 Presto 不支持这种语法。

## Function Vector

在 Velox 中，lambda 函数必须实现为向量函数。 这些接收它们的 lambda 输入作为 FUNCTION 类型的向量。 例如，“filter”函数接收两个向量：一个 ARRAY 类型的向量和另一个 FUNCTION 类型的向量。

函数类型是一种嵌套类型，其子项包含 lambda 参数类型，后跟 lambda 返回类型。 上面“过滤器”函数的 lambda 参数的确切类型是 FUNCTION(INTEGER(), BOOLEAN())。

函数向量是使用 FunctionVector 类实现的。 这些向量以紧凑的形式存储表示可执行 lambda 表达式的可调用对象。 在大多数情况下，所有行的 lambda 表达式都是相同的，但正如我们在上面看到的，不同的行可能与不同的 lambda 相关联。 FunctionVector 存储不同 lambda 的列表以及每个 lambda 适用的一组行。 每个 lambda 都表示为 Callable 类型的对象，它允许在一组行上评估 lambda。

```
class Callable {
  bool hasCapture() const;

  void apply(
      const SelectivityVector& rows,
      BufferPtr wrapCapture,
      exec::EvalCtx* context,
      const std::vector<VectorPtr>& args,
      VectorPtr* result);
};
```

Callable 的“apply”方法类似于 VectorFunction 的“apply”方法，因为它需要一组行来评估，以及一个表示输入数据的向量列表。 例如，“filter”函数使用 Callable::apply 来计算输入数组元素上的 lambda。 在这种情况下，“rows”表示元素向量的行，“args”包含单个元素向量。 “结果”是一个布尔向量，对于通过谓词的元素具有“真”值，对于不通过谓词的元素具有“假”值。

![../_images/lambda-apply.png](https://facebookincubator.github.io/velox/_images/lambda-apply.png)

除了“rows”和“args”之外，Callable::apply() 方法还接受一个可选的“wrapCapture”缓冲区参数。 如果 lambda 表达式使用捕获，则必须指定此参数，例如 如果 Callable::hasCapture() 返回 true。 “wrapCapture”缓冲区用于将顶级捕获行与数组元素或映射键或值的嵌套行对齐。

考虑“filter(a, x -> x >= b)”的例子。 “x >= b”表达式需要两个输入：“x”和“b”。 这里，“x”是数组的一个元素，共有 9 行，而“b”是只有 2 行的顶级列。 为了对齐“x”和“b”，我们需要重复“b”的次数与对应数组中的元素一样多。

![../_images/lambda-apply-with-capture.png](https://facebookincubator.github.io/velox/_images/lambda-apply-with-capture.png)

如果有多个捕获，则所有捕获都需要以相同的方式对齐，例如 它们的值需要重复与相应数组或映射中的元素一样多的次数。 Callable::apply () 中的“wrapCapture”参数用于指定索引缓冲区，该缓冲区可用于将捕获包装在字典向量中以实现这种对齐。 Callable 对象已经包含用于捕获的向量，这些向量不需要包含在“apply()”方法的“args”参数中。

与其他向量不同，FunctionVector 不允许访问单个行的 Callable 对象。 相反，它提供了一个迭代器，它返回 Callable 对象的唯一实例以及它们应用到的一组行。

例如，“filter”函数可以像这样迭代不同的 Callables：

```
auto it = args[1]->asUnchecked<FunctionVector>()->iterator(rows);
while (auto entry = it.next()) {
    ... entry.callable is a pointer to Callable ...
    ... entry.rows is the set of rows this Callable applies to ...
}
```

在大多数情况下，只有一个 Callable 实例，但函数实现需要允许多个。

FunctionVector::iterator() 方法采用 SelectivityVector 参数，该参数将返回的迭代器限制为指定行的子集。 这些通常是评估 lambda 函数的行，例如 VectorFunction::apply() 方法的“rows”参数。

## End-to-End Flow

表达式树中的 lambda 函数调用由具有 LambdaTypedExpr 子节点的 CallTypedExpr 节点表示。 “filter(a, x -> x % 2 = 0)”将表示如下：

![../_images/lambda-end-to-end.png](https://facebookincubator.github.io/velox/_images/lambda-end-to-end.png)

请注意，LambdaTypedExpr 节点没有任何子节点。 表示 lambda 主体的表达式包含在 LambdaTypedExpr 节点内。

该表达式树被编译成可执行表达式树。 LambdaTypedExpr 被编译成一种特殊形式的 LambdaExpr，它包括一个编译体（一个可执行表达式的实例，例如 std::shared_ptr<Expr>）和一个用于捕获的 FieldReference 实例列表。 计算 LambdaExpr 的结果是一个 FunctionVector。 LambdaExpr::evalSpecialForm() 创建 Callable 的实例并将它们存储在 FunctionVector 中。

## Lambda Function Signature

要指定 lambda 函数的签名，请对 lambda 参数的类型使用“function(argType1, argType2,.., returnType)”语法。 以下是“过滤器”函数的签名示例：

```
// array(T), function(T, boolean) -> array(T)
return {exec::FunctionSignatureBuilder()
            .typeVariable("T")
            .returnType("array(T)")
            .argumentType("array(T)")
            .argumentType("function(T,boolean)")
            .build()};
```

## Testing

测试框架不支持 Presto SQL lambda 表达式，例如 不能直接评估“filter(a, x - >x >= b)”表达式。 相反，使用 FunctionBaseTest 类的 registerLambda 辅助方法来注册 lambda 表达式并为其命名，然后使用该名称来指定 lambda 参数。 这是一个在测试中评估“filter(a, x ->x >= b)”表达式的示例：

```
auto rowType = ROW({"a", "b"}, {ARRAY(BIGINT()), BIGINT()});

registerLambda("lambda", ROW({"x"}, {BIGINT()}), rowType, "x >= b"));

auto result =
    evaluate<BaseVector>("filter(a, function('lambda'))", data);
```

registerLambda 的第一个参数是 lambda 的名称。 此名称稍后可用于在函数调用中引用 lambda。

第二个参数是 lambda 的签名，例如 lambda 参数列表及其名称和类型。

第三个参数是整个表达式的输入数据的类型。 这用于解析捕获的类型。

最后一个参数是作为 SQL 表达式的 lambda 主体。

要将 lambda 表达式指定为 lambda 函数的参数，请使用 function ('<lambda-name>') 语法。