# 表达式执行

Velox 具有矢量化表达式执行功能。 它在 FilterProject 运算符中用于执行过滤器和project表达式，并在 HiveConnector 中用于评估“剩余”过滤器表达式。 它也可以单独使用。

查看 velox/example/ExpressionEval.cpp 以获取 API 的示例用法。

## Expression Trees

表达式求值将表达式树作为输入。 树中的每个节点都是 core::ITypedExpr 的子类，它指定返回类型和零个或多个输入表达式（树中的子节点）。 每个表达式可以是以下之一：

- FieldAccessTypedExpr
- ConstantTypedExpr
- CallTypedExpr
- CastTypedExpr
- LambdaTypedExpr

**FieldAccessTypedExpr** 表示输入 RowVector 的列。 该列由名称标识。 这始终是树中的叶节点

**ConstantTypedExpr** 表示一个常量值（或文字）。 这始终是树中的叶节点。

**CallTypedExpr** 表示函数调用。 函数由名称标识。 输入表达式为函数指定参数的数量和类型，从而可以明确地识别特定的函数实现。 该函数可以是简单函数或向量化函数。

CallTypedExpr 还可以通过指定一个预定义名称来表示一种特殊形式。 这些名称不能被简单或向量函数使用。



**and**

AND 表达式。 接受 2 个或更多布尔类型的输入表达式。 如果所有输入都为真，则返回真，如果至少一个输入为假，则返回假，否则返回空。

如果某个输入为另一个输入返回 false 的行生成错误，则该错误将被抑制。

**or**

或表达式。 接受 2 个或更多布尔类型的输入表达式。 如果至少一个输入为真，则返回真，如果所有输入都为假，则返回假，否则返回空。

如果某个输入为另一个输入返回 true 的行生成错误，则该错误将被抑制。

**if**

IF 表达式。 接受 2 或 3 个输入表达式：布尔条件； 然后子句； 可选的 else 子句。

如果未指定 else 子句，则表达式为不通过条件的行返回 null。 当指定 else 子句时，其返回类型必须与 then 子句的返回类型相同。

**switch**

SWITCH 表达式表示 IF 表达式的一般化。 它需要一个或多个（条件，然后子句）对和一个可选的 else 子句

如果未指定 else 子句，则表达式为未通过任何条件的行返回 null。

所有 then 子句和 else 子句的返回类型必须相同。

**cast**

CAST 表达式。 采用单个输入表达式。 结果类型决定了转换的目标类型。

**try**

TRY表达。 采用单个输入表达式。

通过返回相应行的空值来处理输入表达式生成的错误。

**coalesce**

coalesce表达式。 接受多个相同类型的输入表达式。

返回参数列表中的第一个非空值。 与 IF 或 SWITCH 表达式一样，仅在必要时才评估参数。

在评估 AND 和 OR 表达式时，引擎会自适应地重新排序连词，以首先评估最轻量、最具决定性的连词。 例如。 AND 表达式选择评估最常返回 FALSE 的最便宜的合取，而 OR 表达式选择评估最常返回 TRUE 的最便宜的合取。

AND 和 OR 表达式中的错误抑制逻辑旨在提供一致的结果，而不管合取的顺序如何。

**CastTypedExpr** - 与上面的 CAST 表达式相同。

**LambdaTypedExpr** 表示由输入类型（lambda 的签名）和一个表达式（lambda 的主体）指定的 lambda 表达式。 这始终是树中的叶节点。



## Expression Evaluation

表达式评估分为两个步骤：编译和评估。 编译步骤采用 core::ITypedExpr 树形式的表达式列表，并生成 exec::Expr 树形式的可执行表达式列表。 评估将输入数据作为 RowVector，评估编译的表达式并返回结果。 评估步骤可以根据需要重复多次以处理所有数据。

### Compilation

要编译一个表达式，需要创建一个 exec::ExprSet 的实例。 ExprSet 的构造函数接受一个表达式列表（core::ITypedExpr 指向表达式树的根）和一个上下文（core::ExecCtx）。 构造函数处理表达式并创建 exec::Expr 类实例的树。

ExprSet 接受多个表达式并识别所有表达式中的公共子表达式，以便它们可以只计算一次。 FilterProject 运算符受益于此功能，因为它为所有过滤器和项目表达式创建单个 ExprSet。

表达式树中的每个节点都被转换为 exec::Expr 类的相应实例。

| core::ITypedExpr node | exec::Expr instance                                          |
| :-------------------- | :----------------------------------------------------------- |
| FieldAccessTypedExpr  | FieldReference                                               |
| ConstantTypedExpr     | ConstantExpr                                                 |
| CallTypedExpr         | Expr if function name points to a simple or vector function;CastExpr if function name is “cast”;ConjunctExpr if function name is “and” or “or”;SwitchExpr if function name is “if” or “switch”;TryExpr if function name is “try”; |
| CastTypedExpr         | CastExpr                                                     |
| LambdaTypedExpr       | LambdaExpr                                                   |

处理 CallTypedExpr 节点以确定函数名称是否引用特殊形式的表达式或函数（矢量化或简单）。 查找按以下顺序执行，搜索在第一个匹配时停止：

1. 检查名称是否匹配特殊形式之一

2. 检查名称和签名（例如输入类型）是否与简单函数之一匹配

3. 检查名称是否与矢量化函数之一匹配

下图显示了 strpos(upper (a), ‘FOO’) > 0 OR strpos(upper(a), ‘BAR’) > 0 表达式的表达式树。 这里，upper (a) 是一个公共子表达式。 它由在树中出现两次的 Expr 类的单个实例表示。

![../_images/cse.png](https://facebookincubator.github.io/velox/_images/cse.png)



#### Constant Folding

一旦我们有了可执行表达式树 (exec::Expr)，ExprSet 就会识别不依赖于任何列的确定性子表达式，对它们求值并用单个常量表达式替换它们。 这种优化称为常量折叠。

例如，在表达式 upper(a) > upper('Foo') 中，子表达式 upper ('Foo') 是确定性的，不依赖于任何列。 它将在编译期间进行评估，并由单个 ConstantExpr 节点 FOO 替换。

![../_images/constant-folding.png](https://facebookincubator.github.io/velox/_images/constant-folding.png)

#### Flatten ANDs and ORs

相邻的 AND 节点合并为一个。 同样，相邻的 OR 节点合并为一个。 这将在 AND 和 OR 表达式执行期间最大化自适应连接重新排序的效果。

![../_images/flatten-and.png](https://facebookincubator.github.io/velox/_images/flatten-and.png)

#### Expression Metadata

可执行表达式包括一组在评估期间使用的元数据。 这由 Expr::computeMetadata() 虚拟方法计算并存储在 exec::Expr 类的成员变量中。

*distinctFields_* - 如果与父表达式不同，则为不同输入列的列表。 如果不同输入列的列表与父表达式的相同，则此列表为空。

*propagatesNulls_* - 布尔值，指示任何输入列中的 null 是否会导致此表达式始终为该行返回 null。

*deterministic_* -  指示此表达式及其所有子表达式是否具有确定性的布尔值。

*hasConditionals_* - 指示此表达式或其任何子表达式是否为 IF、SWITCH、AND 或 OR 表达式的布尔值。

*isMultiplyReferenced_* - 指示这是否是公共子表达式的布尔值，例如 在 ExprSet 管理的表达式集中多次出现的子表达式。

这是表达式 strpos(upper (a), 'FOO') > 0 OR strpos(upper(b), BAR) > 0 的 distinctFields_ 示例。每个表达式的不同字段显示在表达式右侧的括号中 节点。 OR 节点有 2 个字段：a 和 b。 每个更大的 then 节点都有一个字段：a 或 b。 评估 strpos 和 upper 函数的节点有一个空的不同字段列表，因为它们依赖于与其父节点完全相同的列。 不同的字段元数据触发输入数据的编码剥离，并允许在唯一值的子集上运行整个子表达式。

![../_images/distinct-fields.png](https://facebookincubator.github.io/velox/_images/distinct-fields.png)

### Evaluation

ExprSet 的一个实例代表一个或多个可执行表达式。 可以重复调用 ExprSet::eval() 方法来评估多批数据上的所有表达式或表达式的子集。

FilterProject 运算符对所有过滤器和项目表达式使用 ExprSet 的单个实例。 对于每批输入数据，运算符首先对所有输入行计算过滤器表达式，然后对通过过滤器的行子集计算所有项目表达式。 如果没有行通过过滤器，则跳过项目表达式的评估。

ExprSet::eval() 的输入是 EvalCtx，它包含一个表示输入数据的 RowVector 和一个 SelectivityVector，它标识要评估表达式的行子集。

#### Common SubExpressions (CSEs)

ExprSet::eval() 为单个表达式调用 Expr::eval()。 Expr::eval () 首先检查表达式是否是确定性公共子表达式（isMultiplyReferenced_ == true），如果是，则它是否已经被计算过。 如果是，则返回先前计算的结果并结束评估。

先前评估中使用的行集可能少于当前行集。 在这种情况下，评估继续计算缺失行的表达式。 结果与先前计算的值相结合以产生最终结果。

单个表达式递归调用输入表达式的 Expr::eval()。 这允许公共子表达式优化应用于树的任何级别，而不仅仅是根。

#### Computing on Distinct Values Only

当输入被字典编码时，确定性表达式仅在不同的值上计算。这是通过检查输入列 (distinctFields_) 以识别共享字典包装、剥离这些包装以提取一组内部向量以及一组与原始行相对应的索引、评估这些内部向量上的表达式并将结果包装成使用原始包装的字典向量。

说明这种机制的一种方法是考虑“颜色”列上的上（颜色）表达式，该列是使用 3 个值的字典进行字典编码的：0 - 红色，1 - 绿色，2 - 蓝色。假设字典向量有 1'000 个条目。这些用一个范围为 [0, 2] 的 1000 个值的索引缓冲区和一个大小为 3 的内部平面向量表示：[red, green, blue]。在评估 upper(color) 表达式时，Expr::peelEncodings () 方法用于剥离字典并生成一组新输入：内部平面向量或大小 3 以及该向量的一组索引：[0, 1, 2]。然后，“upper”函数应用于 3 个值 - [red, green, blue] - 以生成另一个大小为 3 的平面向量：[RED, GREEN, BLUE]。最后，使用原始索引将结果包装在字典向量中，以生成以大写形式表示 1000 个颜色值的字典向量。

编码剥离发生在表达式树中取决于给定列集的最高节点处。这是通过将剥离编码方法应用于 distinctFields_ 来实现的，只有当列集与父表达式不同时才会填充该方法。例如。在表达式 f(g(color)) 中，字典编码在表达式树的最顶部被剥离，整个表达式仅在 3 个不同的值上进行评估。

#### Memoizing the Dictionaries

当输入向量来自 TableScan 时，我们可以有多批输入字典向量引用相同的基向量。 “颜色”列可能有数百万行引用相同的基本值集：红色、绿色、蓝色。 在这种情况下，每批输入都有一个字典向量，具有相同的基向量和不同的索引缓冲区。 Expr::eval() 会记住在底层基向量上计算表达式的结果，并将这些结果重新用于后续批次。 对于每个新批次，它只使用输入向量的索引缓冲区包装原始结果。 此逻辑在 Expr::evalWithMemo() 方法中实现，并且仅适用于确定性表达式。

#### Handling Nulls

当表达式传播空值（前面描述的propagatingNulls_ 元数据）时，仅对没有输入为空的行计算表达式，并更新结果以为具有空输入的行设置空值。 在这里，DictionaryVector 允许添加空值就派上用场了（例如 DictionaryVector 有一个空值缓冲区，它与基向量的空值缓冲区分开）。 因此，无论表达式评估的结果是平面编码的还是字典编码的，都可以有效地添加空值。

#### Evaluation Algorithm

表达式求值以深度优先顺序从根开始遍历表达式树。 对每个节点执行一系列操作。

![../_images/expression-evaluation.png](https://facebookincubator.github.io/velox/_images/expression-evaluation.png)

1. Expr::eval - 节点评估的入口点。检查表达式是否是已评估的共享子表达式。如果是这样，请检查它是否针对所有必要的行进行了评估。如果是这样，则产生结果并提前终止评估。否则，将要评估的行设置为缺少结果的行的子集，然后继续下一步。

2. Expr::evalEncodings - 如果表达式是确定性的并且依赖的列少于其父列，请尝试剥离输入列的共享编码。如果剥离成功，则将输入列替换为相应的内部向量，将用于评估的行集更新为内部向量中的相应行，并存储剥离的包装以供以后使用。

3. Expr::evalWithNulls - 如果表达式传播空值，检查输入列并识别至少一个输入为空的行。从行集中删除这些行以进行评估。

4. Expr::evalAll - 表达式可以是特殊形式或函数调用。如果它是特殊形式，则通过调用 Expr::evalSpecialForm() 来计算表达式。如果是函数调用，则通过在子节点上调用 Expr::eval() 递归评估所有输入表达式并生成输入向量。如果函数具有默认的空行为，请识别输入向量为空的所有行，并将这些行从行集中删除以进行评估。如果函数是确定性的并且输入向量不平坦，请尝试剥离编码。如果剥离成功，则将输入向量替换为相应的内部向量，将用于评估的行集更新为内部向量中的相应行，并存储剥离的包装以供以后使用。通过调用 VectorFunction::apply() 来评估函数。通过使用剥离编码包装它们并在由于空输入而被删除的行上设置空值来调整结果。注意：此步骤中对空值的处理和编码剥离似乎与 Expr::evalEncodings 和 Expr::evalWithNulls 中的类似步骤重复。不同之处在于 Expr::evalEncodings 和 Expr::evalWithNulls 处理为整个表达式树提供的输入数据，而此步骤处理通过评估输入表达式计算的输入向量。

5. Finalize - 为由于空输入而从评估中删除的行设置空值。如果任何编码被剥离，使用它来包装结果。如果表达式是共享子表达式，并且之前的评估有部分结果，则将其合并到最终结果中，然后保存结果以供将来使用。

#### Error Handling in AND, OR, TRY

在计算 AND 表达式时，如果某个输入为另一输入返回 false 的行生成错误，则该错误将被抑制。

在评估 OR 表达式时，如果某个输入为另一个输入返回 true 的行生成错误，则该错误将被抑制。

AND 和 OR 表达式中的错误抑制逻辑旨在提供一致的结果，而不管合取的顺序如何。

TRY 表达式通过将相应行中的结果设置为 null 来处理异常。 AND、OR 和 TRY 表达式中的错误处理依赖于所有表达式和向量函数来正确支持 EvalCtx::throwOnError 标志。 当设置为 false 时，如果行的数据无效，表达式和向量函数不应抛出异常，而是通过调用 EvalCtx::setError(row, exception) 记录错误。



#### Adaptive Conjunct Reordering in AND and OR

AND 和 OR 表达式使用与选择性 ORC 读取器相同的机制来跟踪单个连接的性能并自适应地重新排序这些连接以评估最先删除大多数行的连接。

AND 表达式首先计算最常返回 FALSE 的最便宜的合取，而 OR 表达式首先计算最常返回 TRUE 的最便宜的合取。



#### Evaluation of IF, SWITCH

SWITCH 表达式评估经过以下步骤： * 评估所有行的第一个条件。 * 在第一个条件为真的行子集上评估第一个“then”子句，并生成部分填充的结果向量。 * 在第一个条件不成立的行上评估第二个条件。 * 在第二个条件所在的行子集上评估第二个“then”子句。 在评估“then”子句时将部分填充的结果向量传递给 Expr::eval，并期望表达式使用指定行的计算值更新结果向量，同时保留已计算的结果。 * 继续处理所有的（条件，然后子句）对。 如果行数用完，请提前终止。 * 最后，评估剩余行的 else 子句。 如果未指定 else 子句，则为其余行设置空值。

SWITCH 表达式将 EvalCtx::isFinalSelection 标志设置为 false。 表达式应使用此标志来决定是否必须保留部分填充的结果向量或可以覆盖。
