# 如何增加一个聚合函数？

聚合函数由 HashAggregation 运算符计算。 单个运算符中可以有一个或多个聚合函数。 这里有些例子。

```
SELECT count(*) FROM t
```

```
SELECT count(*), sum(b) FROM t
```

```
SELECT a, count(*), sum(b), sum(c) FROM t GROUP BY 1
```

```
SELECT a, b, min(c) FROM t GROUP BY 1, 2
```

通常，聚合分两步计算：部分聚合和最终聚合。

部分聚合采用原始数据并产生中间结果。 最终聚合采用中间结果并产生最终结果。 在某些情况下还使用单一和中间聚合。 当数据已经在分组键上分区时使用单一聚合，因此不需要洗牌。 中间聚合用于将在多个线程中并行计算的部分聚合的结果进行组合，以减少发送到最终聚合阶段的数据量。

聚合的四种类型（步骤）仅通过输入和输出的类型来区分：

| Step             | Input                | Output               |
| :--------------- | :------------------- | :------------------- |
| **Partial**      | Raw Data             | Intermediate Results |
| **Final**        | Intermediate Results | Final Results        |
| **Single**       | Raw Data             | Final Results        |
| **Intermediate** | Intermediate Results | Intermediate Results |

在某些情况下，部分聚合和最终聚合执行的计算是相同的。 sum()、min() 和 max() 聚合就是这种情况。 在大多数情况下，它们是不同的。 例如，partial count() 聚合对传入值进行计数，而 final :func:count 聚合将部分计数相加以产生总数。

聚合函数的签名由原始输入数据的类型、中间结果的类型和最终结果的类型组成。

## Memory Layout

HashAggregation 运算符将数据存储在行中。 每行对应于分组键值的唯一组合。 全局聚合将数据存储在一行中。

聚合函数可以按其累加器的类型分为三组：

- - Fixed width accumulators:

    [`count()`](https://facebookincubator.github.io/velox/functions/aggregate.html#id0), [`sum()`](https://facebookincubator.github.io/velox/functions/aggregate.html#sum), [`avg()`](https://facebookincubator.github.io/velox/functions/aggregate.html#avg)[`min()`](https://facebookincubator.github.io/velox/functions/aggregate.html#min), [`max()`](https://facebookincubator.github.io/velox/functions/aggregate.html#max), [`arbitrary()`](https://facebookincubator.github.io/velox/functions/aggregate.html#arbitrary) (for fixed-width types)

- - Variable width accumulators with append-only semantics:

    [`array_agg()`](https://facebookincubator.github.io/velox/functions/aggregate.html#array_agg)[`map_agg()`](https://facebookincubator.github.io/velox/functions/aggregate.html#map_agg)

- - Variable width accumulators which can be modified in any way, not just appended to.

    [`min()`](https://facebookincubator.github.io/velox/functions/aggregate.html#min), [`max()`](https://facebookincubator.github.io/velox/functions/aggregate.html#max) (for strings)[`arbitrary()`](https://facebookincubator.github.io/velox/functions/aggregate.html#arbitrary) (for variable-width types)[`approx_percentile()`](https://facebookincubator.github.io/velox/functions/aggregate.html#id4)[`approx_distinct()`](https://facebookincubator.github.io/velox/functions/aggregate.html#id1)

累加器的固定宽度部分存储在行中。 可变宽度部分（如果存在）使用 HashStringAllocator 分配，并且指针存储在固定宽度部分中。

行是一个连续的字节缓冲区。 给定 N 个聚合，前 N / 8 个字节存储空标志，每个聚合一个位，然后是固定宽度的累加器。

![../_images/aggregation-layout.png](https://facebookincubator.github.io/velox/_images/aggregation-layout.png)

## Aggregate class

添加聚合函数：

- Prepare:
  - 弄清楚输入、中间和最终类型是什么
  - 弄清楚什么是部分计算和最终计算
  - 设计accumulator.
  - 创建一个扩展 velox::exec::Aggregate 基类的新类（参见 velox/exec/Aggregate.h）并实现虚拟方法。
- 使用 exec::registerAggregateFunction(…) 注册新函数。
- 写测试.
- 写文档.



## Accumulator size

velox::exec::Aggregate 接口的实现可以从 accumulatorFixedWidthSize() 方法开始。

```
// Returns the fixed number of bytes the accumulator takes on a group
// row. Variable width accumulators will reference the variable
// width part of the state from the fixed part.
virtual int32_t accumulatorFixedWidthSize() const = 0;
```

HashAggregation 运算符在初始化期间使用此方法来计算行的总大小并计算出不同聚合将存储其数据的偏移量。 运算符然后为每个聚合调用 velox::exec::Aggregate::setOffsets 方法来指定累加器的位置。

```
// Sets the offset and null indicator position of 'this'.
// @param offset Offset in bytes from the start of the row of the accumulator
// @param nullByte Offset in bytes from the start of the row of the null flag
// @param nullMask The specific bit in the nullByte that stores the null flag
void setOffsets(int32_t offset, int32_t nullByte, uint8_t nullMask)
```

基类通过将偏移量存储在成员变量中来实现 setOffsets 方法。

```
// Byte position of null flag in group row.
int32_t nullByte_;
uint8_t nullMask_;
// Offset of fixed length accumulator state in group row.
int32_t offset_;
```

通常，聚合函数不直接使用偏移量。 相反，它使用基类中的辅助方法。

访问累加器：

```
template <typename T>
T* value(char* group) const {
  return reinterpret_cast<T*>(group + offset_);
}
```

操作空标志：

```
bool isNull(char* group) const;

// Sets null flag for all specified groups to true.
// For any given group, this method can be called at most once.
void setAllNulls(char** groups, folly::Range<const vector_size_t*> indices);

inline bool clearNull(char* group);
```

## Initialization

一旦你有了 accumulatorFixedWidthSize()，下一个实现的方法是 initializeNewGroups()。

```
// Initializes null flags and accumulators for newly encountered groups.
// @param groups Pointers to the start of the new group rows.
// @param indices Indices into 'groups' of the new entries.
virtual void initializeNewGroups(
    char** groups,
    folly::Range<const vector_size_t*> indices) = 0;
```

HashAggregation 运算符每次遇到分组键的新组合时都会调用此方法。 此方法应初始化新组的累加器。 例如，部分“count”和“sum”聚合会将累加器设置为零。 许多聚合函数会通过调用 exec::Aggregate::setAllNulls(groups, indices) 辅助方法将 null 标志设置为 true。

## GroupBy aggregation

此时您已经实现了 accumulatorFixedWidthSize() 和 initializeNewGroups() 方法。 现在，我们可以继续实现端到端分组聚合。 我们需要以下几部分：

1. 将原始输入添加到部分累加器的逻辑：addRawInput() 函数。
2. 从部分累加器产生中间结果的逻辑：extractAccumulators() 函数.
3. 将中间结果添加到最终累加器的逻辑：addIntermediateResults() 方法。
4. 从最终累加器产生最终结果的逻辑：extractValues() 函数

我们从 addRawInput() 方法开始，它接收原始输入向量并将数据添加到部分累加器：

```
// Updates the accumulator in 'groups' with the values in 'args'.
// @param groups Pointers to the start of the group rows. These are aligned
// with the 'args', e.g. data in the i-th row of the 'args' goes to the i-th group.
// The groups may repeat if different rows go into the same group.
// @param rows Rows of the 'args' to add to the accumulators. These may not be
// contiguous if the aggregation is configured to drop null grouping keys.
// This would be the case when aggregation is followed by the join on the
// grouping keys.
// @param args Data to add to the accumulators.
// @param mayPushdown True if aggregation can be pushdown down via LazyVector.
// The pushdown can happen only if this flag is true and 'args' is a single
// LazyVector.
virtual void addRawInput(
    char** groups,
    const SelectivityVector& rows,
    const std::vector<VectorPtr>& args,
    bool mayPushdown = false) = 0;
```

addRawInput() 方法将使用 DecodedVector 解码输入数据。 然后，遍历行以更新累加器。 我建议为每个输入向量定义一个 DecodedVector 类型的成员变量。 这允许在输入批次之间重用解码输入所需的内存。

在实现 addRawInput() 方法之后，我们继续添加用于提取中间结果的逻辑。

```
// Extracts partial results (used for partial and intermediate aggregations).
// @param groups Pointers to the start of the group rows.
// @param numGroups Number of groups to extract results from.
// @param result The result vector to store the results in.
virtual void
extractAccumulators(char** groups, int32_t numGroups, VectorPtr* result) = 0;
```

接下来，我们实现 addIntermediateResults() 方法，该方法接收中间结果并更新最终累加器。

```
virtual void addIntermediateResults(
    char** groups,
    const SelectivityVector& rows,
    const std::vector<VectorPtr>& args,
    bool mayPushdown = false) = 0;
```

最后，我们实现了从累加器中提取最终结果的 extractValues() 方法。

```
// Extracts final results (used for final and single aggregations).
// @param groups Pointers to the start of the group rows.
// @param numGroups Number of groups to extract results from.
// @param result The result vector to store the results in.
virtual void
extractValues(char** groups, int32_t numGroups, VectorPtr* result) = 0;
```

GroupBy 聚合代码路径完成。 我们继续进行全局聚合

## Global aggregation

全局聚合类似于分组聚合，但只有一个组和一个累加器。 实现分组聚合后，启用全局聚合唯一需要做的就是实现 addSingleGroupRawInput() 和 addSingleGroupIntermediateResults() 方法。

```
// Updates the single accumulator used for global aggregation.
// @param group Pointer to the start of the group row.
// @param allRows A contiguous range of row numbers starting from 0.
// @param args Data to add to the accumulators.
// @param mayPushdown True if aggregation can be pushdown down via LazyVector.
// The pushdown can happen only if this flag is true and 'args' is a single
// LazyVector.
virtual void addSingleGroupRawInput(
    char* group,
    const SelectivityVector& allRows,
    const std::vector<VectorPtr>& args,
    bool mayPushdown) = 0;
```

## Factory function

我们现在可以编写一个工厂函数来创建新聚合函数的实例并通过调用 exec::registerAggregateFunction (...) 并指定函数名称和签名来注册它。

HashAggregation 运算符使用此函数创建聚合函数的实例。 为每个执行线程创建一个新实例。 当部分聚合在 5 个线程上运行时，它使用每个聚合函数的 5 个实例。

工厂函数采用 core::AggregationNode::Step (partial/final/intermediate/single)，它告诉期望输入的类型、输入类型和结果类型。

```
bool registerApproxPercentile(const std::string& name) {
  std::vector<std::shared_ptr<exec::AggregateFunctionSignature>> signatures;
  ...

  exec::registerAggregateFunction(
      name,
      std::move(signatures),
      [name](
          core::AggregationNode::Step step,
          const std::vector<TypePtr>& argTypes,
          const TypePtr& resultType) -> std::unique_ptr<exec::Aggregate> {
        if (step == core::AggregationNode::Step::kIntermediate) {
          return std::make_unique<ApproxPercentileAggregate<double>>(
              false, false, VARBINARY());
        }

        auto hasWeight = argTypes.size() == 3;
        TypePtr type = exec::isRawInput(step) ? argTypes[0] : resultType;

        switch (type->kind()) {
          case TypeKind::BIGINT:
            return std::make_unique<ApproxPercentileAggregate<int64_t>>(
                hasWeight, resultType);
          case TypeKind::DOUBLE:
            return std::make_unique<ApproxPercentileAggregate<double>>(
                hasWeight, resultType);
          ...
        }
      });
  return true;
}

static bool FB_ANONYMOUS_VARIABLE(g_AggregateFunction) =
    registerApproxPercentile(kApproxPercentile);
```

使用 FunctionSignatureBuilder 创建描述受支持签名的 FunctionSignature 实例。 每个签名包括零个或多个输入类型、中间结果类型和最终结果类型。

FunctionSignatureBuilder 和 FunctionSignature 支持类似 Java 的泛型、可变数量的参数和 lambda。 这是 approx_percentile() 函数的签名示例。 该函数采用数值类型的值参数、INTEGER 类型的可选权重参数和 DOUBLE 类型的百分比参数。 中间类型不依赖于输入类型，并且始终为 VARBINARY。 最终结果类型与输入值类型相同。

```
for (const auto& inputType :
       {"tinyint", "smallint", "integer", "bigint", "real", "double"}) {
    // (x, double percentage) -> varbinary -> x
    signatures.push_back(exec::AggregateFunctionSignatureBuilder()
                             .returnType(inputType)
                             .intermediateType("varbinary")
                             .argumentType(inputType)
                             .argumentType("double")
                             .build());

    // (x, integer weight, double percentage) -> varbinary -> x
    signatures.push_back(exec::AggregateFunctionSignatureBuilder()
                             .returnType(inputType)
                             .intermediateType("varbinary")
                             .argumentType(inputType)
                             .argumentType("bigint")
                             .argumentType("double")
                             .build());
  }
```

## Testing

是时候将所有部分放在一起并测试新功能的工作情况了。

使用 velox/aggregates/tests/AggregationTestBase.h 中的 AggregationTestBase 作为测试的基类。

如果 DuckDB 支持新的聚合函数，您可以使用 DuckDB 来查看结果。 在这种情况下，您编写一个带有使用新函数的聚合节点的查询计划，并将该计划的结果与在 DuckDB 上运行的 SQL 查询进行比较。

```
agg = PlanBuilder()
          .values(vectors)
          .partialAggregation({0}, {"sum(c1)"})
          .finalAggregation()
          .planNode();
assertQuery(agg, "SELECT c0, sum(c1) FROM tmp GROUP BY 1");
```

如果 DuckDB 不支持新功能，则需要手动指定预期结果。

```
void testGroupByAgg(
    const VectorPtr& keys,
    const VectorPtr& values,
    double percentile,
    const RowVectorPtr& expectedResult) {
  auto rowVector = makeRowVector({keys, values});

  auto op =
      PlanBuilder()
          .values({rowVector})
          .singleAggregation(
              {0}, {fmt::format("approx_percentile(c1, {})", percentile)})
          .planNode();

  assertQuery(op, expectedResult);

  op = PlanBuilder()
           .values({rowVector})
           .partialAggregation(
               {0}, {fmt::format("approx_percentile(c1, {})", percentile)})
           .finalAggregation()
           .planNode();

  assertQuery(op, expectedResult);
}
```

如上例所示，您可以使用 PlanBuilder 创建聚合计划节点。

- partialAggregation() 为部分聚合创建计划节点
- finalAggregation() 为最终聚合创建计划节点
- singleAggregation() 为单个 agg 创建计划节点

partialAggregation() 和 singleAggregation() 方法采用分组键列表（作为输入 RowVector 的索引）和聚合函数列表（作为 SQL 字符串）。 测试框架解析聚合函数的 SQL 字符串，并使用存储在注册表中的签名推断结果的类型。

finalAggregation() 方法不接受任何参数，并从前面的部分聚合中推断出分组键和聚合函数。

在构建包含最终聚合但没有相应部分聚合的计划时，请使用 finalAggregation() 变体，该变体采用分组键、聚合函数和聚合函数结果类型。

```
PlanBuilder()
    ...
    .finalAggregation({0}, {"approx_percentile(a0)"}, {DOUBLE()})
```

使用具有 DOUBLE 结果类型的 approx_percentile 聚合函数创建最终聚合。



## Function names

与标量函数相同，聚合函数名称不区分大小写。 注册函数并为给定表达式解析时，名称会自动转换为小写。



## Accumulator

可变宽度累加器需要使用 HashStringAllocator 来分配内存。 分配器的一个实例在基类中可用：velox::exec::Aggregate::allocator_。

有时您需要创建自定义累加器。 有时，现有的累加器之一会完成这项工作。

min()、max() 和 absolute() 函数使用的 SingleValueAccumulator 可用于存储可变宽度类型的单个值，例如 字符串、数组、映射或结构。

array_agg() 和 map_agg() 使用的 ValueList 累加器会累加一个值列表。 这是一个只能追加的累加器。

在 velox/exec/HashStringAllocator.h 中定义的 StlAllocator 可用于制作由 HashStringAllocator 分配的内存支持的 STL 容器（例如 std::vector）。 StlAllocator 本身不是累加器，但可用于设计使用 STL 容器的累加器。 它由 approx_percentile() 和 approx_distinct() 使用。

从 HashStringAllocator 分配的内存需要在 destroy() 方法中释放。 有关示例，请参见 velox/aggregates/ArrayAgg.cpp。

```
void destroy(folly::Range<char**> groups) override {
  for (auto group : groups) {
    if (auto header = value<ArrayAccumulator>(group)->elements.begin()) {
      allocator_->free(header);
    }
  }
}
```

## End-to-End Testing

要确认聚合函数作为查询的一部分端到端工作，请更新 presto_cpp 存储库中 TestHiveAggregationQueries.java 中的 testAggregations() 测试以添加使用新函数的查询。

```
assertQuery("SELECT orderkey, array_agg(linenumber) FROM lineitem GROUP BY 1");
```