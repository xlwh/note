# Plan Nodes and Operators

Velox 查询计划是 PlanNode 的树。 每个 PlanNode 有零个或多个子 PlanNode。 为了执行查询计划，Velox 将其转换为一组管道。 每个管道由与计划的线性子树相对应的线性运算符序列组成。 通过将除一个子节点之外的所有子节点与具有两个或多个子节点的每个节点断开连接，将计划树分解为一组线性子树。

![../_images/local-planner.png](https://facebookincubator.github.io/velox/_images/local-planner.png)

计划节点到算子的转换大多是一对一的。 一些例外是：

- Filter 节点后跟 Project 节点转换为单个算子 FilterProject
- 具有两个或多个子节点的节点将转换为多个运算符，例如 HashJoin 节点转换为一对运算符：HashProbe 和 HashBuild。

叶节点对应的算子称为源算子。 只有计划节点的子集可以位于计划树的叶子上。 这些是：

- TableScanNode
- ValuesNode
- ExchangeNode
- MergeExchangeNode

这是支持的计划节点和相应运算符的列表:

| Plan Node             | Operator(s)                             | Leaf Node / Source Operator |
| :-------------------- | :-------------------------------------- | :-------------------------- |
| TableScanNode         | TableScan                               | Y                           |
| FilterNode            | FilterProject                           |                             |
| ProjectNode           | FilterProject                           |                             |
| AggregationNode       | HashAggregation or StreamingAggregation |                             |
| GroupIdNode           | GroupId                                 |                             |
| HashJoinNode          | HashProbe and HashBuild                 |                             |
| MergeJoinNode         | MergeJoin                               |                             |
| CrossJoinNode         | CrossJoinProbe and CrossJoinBuild       |                             |
| OrderByNode           | OrderBy                                 |                             |
| TopNNode              | TopN                                    |                             |
| LimitNode             | Limit                                   |                             |
| UnnestNode            | Unnest                                  |                             |
| TableWriteNode        | TableWrite                              |                             |
| PartitionedOutputNode | PartitionedOutput                       |                             |
| ExchangeNode          | Exchange                                | Y                           |
| MergeExchangeNode     | MergeExchange                           | Y                           |
| ValuesNode            | Values                                  | Y                           |
| LocalMergeNode        | LocalMerge                              |                             |
| LocalPartitionNode    | LocalPartition and LocalExchange        |                             |
| EnforceSingleRowNode  | EnforceSingleRow                        |                             |
| AssignUniqueIdNode    | AssignUniqueId                          |                             |

## Plan Nodes

### TableScanNode

表扫描操作从连接器读取数据。 例如，当与 HiveConnector 一起使用时，表扫描从 ORC 或 Parquet 文件中读取数据。

| Property    | Description                                                  |
| :---------- | :----------------------------------------------------------- |
| outputType  | A list of output columns. This is a subset of columns available in the underlying table. The order of columns may not match the schema of the table. |
| tableHandle | Connector-specific description of the table. May include a pushed down filter. |
| assignments | Connector-specific mapping from the table schema to output columns. |

### FilterNode

过滤器操作根据布尔过滤器表达式从输入数据中删除一条或多条记录。

| Property | Description                |
| :------- | :------------------------- |
| filter   | Boolean filter expression. |

### ProjectNode

项目操作根据数据集的输入生成一个或多个附加表达式。 项目操作也可以删除一个或多个输入列。

| Property    | Description                              |
| :---------- | :--------------------------------------- |
| names       | Column names for the output expressions. |
| expressions | Expressions for the output columns.      |

### AggregationNode

聚合操作将输入数据分组到一组分组键上，计算分组键的每个组合的每个度量。

| Property         | Description                                                  |
| :--------------- | :----------------------------------------------------------- |
| step             | Aggregation step: partial, final, intermediate, single.      |
| groupingKeys     | Zero or more grouping keys.                                  |
| preGroupedKeys   | A subset of the grouping keys on which the input is known to be pre-grouped, i.e. all rows with a given combination of values of the pre-grouped keys appear together one after another. The input is not assumed to be sorted on the pre-grouped keys. If input is pre-grouped on all grouping keys the execution will use the StreamingAggregation operator. |
| aggregateNames   | Names for the output columns for the measures.               |
| aggregates       | Expressions for computing the measures, e.g. count(1), sum(a), avg(b). Expressions must be in the form of aggregate function calls over input columns directly, e.g. sum(c) is ok, but sum(c + d) is not. |
| aggregationMasks | For each measure, an optional boolean input column that is used to mask out rows for this particular measure. |
| ignoreNullKeys   | A boolean flag indicating whether the aggregation should drop rows with nulls in any of the grouping keys. Used to avoid unnecessary processing for an aggregation followed by an inner join on the grouping keys. |

### GroupIdNode

为每个指定的分组键集复制输入。 用于在分组集上实现聚合。

| Property               | Description                                                  |
| :--------------------- | :----------------------------------------------------------- |
| groupingSets           | List of grouping key sets. Keys within each set must be unique, but keys can repeat across the sets. |
| outputGroupingKeyNames | Output names for the grouping key columns.                   |
| aggregationInputs      | Input columns to duplicate.                                  |
| groupIdName            | The name for the group-id column that identifies the grouping set. Zero-based integer corresponding to the position of the grouping set in the ‘groupingSets’ list. |

### HashJoinNode and MergeJoinNode

连接操作基于连接表达式将两个单独的输入组合成一个输出。 连接的一个常见子类型是等式连接，其中连接表达式被限制为连接的两个输入之间的等式（或等式 + 空等式）条件列表。

HashJoinNode 表示一种实现，它首先将连接右侧的所有行加载到哈希表中，然后流式连接连接的左侧，探测哈希表以查找匹配行并发出结果。

MergeJoinNode 表示一个实现，它假设两个输入都按连接键排序，并且流式连接双方都在寻找匹配的行并发出结果。

| Property   | Description                                                  |
| :--------- | :----------------------------------------------------------- |
| joinType   | Join type: inner, left, right, full, semi, anti. You can read about different join types in this [blog post](https://dataschool.com/how-to-teach-people-sql/sql-join-types-explained-visually/). |
| leftKeys   | Columns from the left hand side input that are part of the equality condition. At least one must be specified. |
| rightKeys  | Columns from the right hand side input that are part of the equality condition. At least one must be specified. The number and order of the rightKeys must match the number and order of the leftKeys. |
| filter     | Optional non-equality filter expression that may reference columns from both inputs. |
| outputType | A list of output columns. This is a subset of columns available in the left and right inputs of the join. The columns may appear in different order than in the input. |

### CrossJoinNode

交叉连接操作通过将左侧输入的每一行与右侧输入的每一行组合起来，将两个单独的输入组合成一个输出。 如果左输入有 N 行，右输入有 M 行，则交叉连接的输出将包含 N * M 行。

| Property   | Description                                                  |
| :--------- | :----------------------------------------------------------- |
| outputType | A list of output columns. This is a subset of columns available in the left and right inputs of the join. The columns may appear in different order than in the input. |

### OrderByNode

排序或排序依据操作基于一个或多个识别的排序字段以及排序顺序对数据集重新排序。

| Property      | Description                                                  |
| :------------ | :----------------------------------------------------------- |
| sortingKeys   | List of one of more input columns to sort by.                |
| sortingOrders | Sorting order for each of the soring keys. The supported orders are: ascending nulls first, ascending nulls last, descending nulls first, descending nulls last. |
| isPartial     | Boolean indicating whether the sort operation processes only a portion of the dataset. |

### TopNNode

top-n 操作根据一个或多个已识别的排序字段以及排序顺序对数据集进行重新排序。 top-n 不会对整个数据集进行排序，而是只维护确保有限输出所需的记录总数。 top-n 是逻辑排序和逻辑限制操作的组合。

| Property      | Description                                                  |
| :------------ | :----------------------------------------------------------- |
| sortingKeys   | List of one of more input columns to sort by.                |
| sortingOrders | Sorting order for each of the soring keys. See OrderBy for the list of supported orders. |
| count         | Maximum number of rows to return.                            |
| isPartial     | Boolean indicating whether the operation processes only a portion of the dataset. |

### LimitNode

限制操作跳过指定数量的输入行，然后保留指定数量的行并删除其余行。

| Property  | Description                                                  |
| :-------- | :----------------------------------------------------------- |
| offset    | Number of rows of input to skip.                             |
| count     | Maximum number of rows to return.                            |
| isPartial | Boolean indicating whether the operation processes only a portion of the dataset. |

### UnnestNode

unnest 操作将数组和映射扩展为单独的列。 数组扩展为单列，映射扩展为两列（键、值）。 可用于扩展多个列。 在这种情况下，产生与最高基数数组或映射一样多的行（其他列用空值填充）。 可以选择生成一个序数列，该列指定从 1 开始的行号。

| Property           | Description                                                  |
| :----------------- | :----------------------------------------------------------- |
| replicateVariables | Input columns that are returned unmodified.                  |
| unnestVariables    | Input columns of type array or map to expand.                |
| unnestNames        | Names to use for expanded columns. One name per array column. Two names per map column. |
| ordinalityName     | Optional name for the ordinality column.                     |

### TableWriteNode

表写入操作消耗一个输出并通过连接器将其写入存储。 一个例子是编写 ORC 或 Parquet 文件。 表写入操作返回单行，单列包含写入存储的行数。

| Property          | Description                                                  |
| :---------------- | :----------------------------------------------------------- |
| columns           | A list of input columns to write to storage. This may be a subset of the input columns in different order. |
| columnNames       | Column names to use when writing to storage. These can be different from the input column names. |
| insertTableHandle | Connector-specific description of the destination table.     |
| outputType        | An output column containing a number of rows written to storage. |

### PartitionedOutputNode

分区输出操作基于零个或多个分布字段重新分布数据

| Property                 | Description                                                  |
| :----------------------- | :----------------------------------------------------------- |
| keys                     | Zero or more input fields to use for calculating a partition for each row. |
| numPartitions            | Number of partitions to split the data into.                 |
| broadcast                | Boolean flag indicating whether all rows should be sent to all partitions. |
| replicateNullsAndAny     | Boolean flag indicating whether rows with nulls in the keys should be sent to all partitions and, in case there are no such rows, whether a single arbitrarily chosen row should be sent to all partitions. Used to provide global-scope information necessary to implement anti join semantics on a single node. |
| partitionFunctionFactory | Factory to make partition functions to use when calculating partitions for input rows. |
| outputType               | A list of output columns. This is a subset of input columns possibly in a different order. |

### ValuesNode

values 操作返回指定的数据。

| Property | Description            |
| :------- | :--------------------- |
| values   | Set of rows to return. |

### ExchangeNode

以任意顺序合并多个流的接收操作。 输入流来自远程交换或洗牌。

| Property | Description                             |
| :------- | :-------------------------------------- |
| type     | A list of columns in the input streams. |

### MergeExchangeNode

合并多个有序流以保持有序性的接收操作。 输入流来自远程交换或洗牌。

| Property      | Description                                                  |
| :------------ | :----------------------------------------------------------- |
| type          | A list of columns in the input streams.                      |
| sortingKeys   | List of one of more input columns to sort by.                |
| sortingOrders | Sorting order for each of the soring keys. See OrderBy for the list of supported orders. |

### LocalMergeNode

合并多个有序流以保持有序性的操作。 输入流来自本地交换。

| Property      | Description                                                  |
| :------------ | :----------------------------------------------------------- |
| sortingKeys   | List of one of more input columns to sort by.                |
| sortingOrders | Sorting order for each of the soring keys. See OrderBy for the list of supported orders. |

### LocalPartitionNode

将输入数据划分为多个流或将来自多个流的数据组合为单个流的本地交换操作。

| Property                 | Description                                                  |
| :----------------------- | :----------------------------------------------------------- |
| Type                     | Type of the exchange: gather or repartition.                 |
| partitionFunctionFactory | Factory to make partition functions to use when calculating partitions for input rows. |
| outputType               | A list of output columns. This is a subset of input columns possibly in a different order. |

### EnforceSingleRowNode

强制单行操作检查输入是否最多包含一行并返回未修改的行。 如果输入为空，则返回所有值都设置为空的单行。 如果输入包含多于一行，则会引发异常。

用于具有不相关子查询的查询。

### AssignUniqueIdNode

分配唯一 id 操作在输入列的末尾添加一列，每行具有唯一值。 此唯一值将每个输出行标记为在此运算符的所有输出行中唯一。

64 位唯一 ID 以下列方式构建： - 前 24 位 - 任务唯一 ID - 后 40 位 - 操作员计数器值

添加任务唯一 ID 以确保生成的 ID 在分布式查询执行中执行相同查询阶段的所有节点中是唯一的。

| Property     | Description                                                  |
| :----------- | :----------------------------------------------------------- |
| idName       | Column name for the generated unique id column.              |
| taskUniqueId | A 24-bit integer to uniquely identify the task id across all the nodes. |

## Examples

### Join

带有连接的查询计划包括一个 HashJoinNode。 这样的计划被翻译成两个管道：构建和探测。 构建管道正在处理来自连接的构建端的输入，并使用 HashBuild 运算符来构建哈希表。 探测管道正在处理来自连接的探测端的输入，探测哈希表并生成匹配连接条件的行。 构建管道通过一种称为 JoinBridge 的特殊机制向探测管道提供哈希表。 JoinBridge 就像一个未来，其中 HashBuild 运算符使用 HashTable 作为结果完成未来，而 HashProbe 运算符在未来完成时接收 HashTable。

每个管道可以以不同级别的并行度运行。 在下面的示例中，探测管道在 2 个线程上运行，而构建管道在 3 个线程上运行。 当构建管道运行多线程时，每个管道处理构建端输入的一部分。 最后一个完成处理的管道负责组合来自其他管道的哈希表并将最终表发布到 JoinBridge。 当右外连接的探测管道运行多线程时，最后一个完成处理的管道负责从构建端发出与连接条件不匹配的行。

![../_images/join.png](https://facebookincubator.github.io/velox/_images/join.png)



### Local Exchange

本地交换操作有多种用途。 它用于将数据处理的并行性从多线程更改为单线程，反之亦然。 例如，本地交换可用于排序操作，其中部分排序在多线程中运行，然后在单个线程上合并结果。 本地交换操作也用于组合多个管道的结果。 例如，组合 UNION 或 UNION ALL 的多个输入。

举例：

本地交换，可用于组合部分排序的结果以进行最终合并排序。

![../_images/local-exchange-N-to-1.png](https://facebookincubator.github.io/velox/_images/local-exchange-N-to-1.png)



本地交换以在必须运行单线程的操作之后增加并行度

![../_images/local-exchange-1-to-N.png](https://facebookincubator.github.io/velox/_images/local-exchange-1-to-N.png)



用于组合来自多个管道的数据的本地交换，例如 为联合所有

![../_images/local-exchange.png](https://facebookincubator.github.io/velox/_images/local-exchange.png)









