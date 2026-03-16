# Trino Sql优化:Removing redundant ORDER BY

一些SQL构造（如ORDER BY）在许多情况下不会影响查询结果，并且会产生影响性能的负面影响(查询中的每个ORDER BY子句都代表一个排序执行计划)。如果用户无意中在没有效果的地方使用ORDER BY，可能会导致严重的性能下降和资源浪费。

sql规范(ISO 9075 Part 2)中说明:
-  一个`<query expression>`可以包含一个可选的`<order by clause>`。
- `<query expression>`z中行的顺序仅由`<query expression>`直接包含的`<order by clause>`指定。

上述规范意味着，查询引擎可以自由地忽略任何不适合上下文的ORDER BY子句。

<!--more-->

举个例子:

```sql
INSERT INTO some_table 
SELECT * FROM another_table 
ORDER BY field
```

SQL中的表本质上是无序的。一旦数据被写入，就不能保证它在读取时会被排序（对于像Presto这样的并行分布式查询引擎来说尤其如此）。在数据插入期间。执行ORDER BY会导致执行性能不佳，因为分布式排序的合并步骤减少了查询的并行性，且排序消耗更多的CPU和内存。

```sql
SELECT *
FROM some_table 
   JOIN (SELECT * FROM another_table ORDER BY field) u 
   ON some_table.key = u.key 
```

在这种情况下，join中涉及的表是否被排序并不重要，因为Presto将从其中一个构建哈希查找表以执行join操作。保留ORDER BY只会导致查询执行不佳。

**ORDER BY什么时候生效**？

只有属于同一`<query expression>`的操作受ORDER BY 影响。查询表达式是具有以下结构的块：

```bnf
<query expression> ::=
  [ <with clause> ] 
  <query expression body>
  [ <order by clause> ] 
  [ <result offset clause> ] 
  [ <fetch first clause> ]
```

其中`<query expression body>`可以被细分为一个操作符(UNION、INTERSECT、EXCEPT)， 一个SELECT构造、VALUES或TABLE子句。

在ORDER BY之后发生的唯一操作是FETCH FIRST（又名LIMIT）和OFFSET。 除非子查询包含这两个子句之一，否则查询引擎可以自由删除ORDER BY 子句不违反规范规定的语义学。例如，在下面的子句中ORDER BY是生效的。

```sql
SELECT *
FROM some_table
WHERE field = (
    SELECT a 
    FROM another_table 
    ORDER BY b 
    LIMIT 1)
```

其他数据库以各种方式解决这个问题。MariaDB 和Hive 3.0 将忽略冗余的ORDER BY子句。SQL Server将产生错误：

```
The ORDER BY clause is invalid in views, inline functions, derived tables, subqueries, and common table expressions, unless TOP or FOR XML is also specified.
```

用户在使用子查询时常常隐式的依赖子查询的顺序：

```sql
SELECT *
FROM (
    SELECT *
    FROM nation
    ORDER BY name DESC
)
limit 10;
```

正确方法是:

```sql
SELECT *
FROM (
    SELECT *
    FROM nation
)
ORDER BY name DESC
limit 10;
```

对于聚合函数和窗口函数:

```sql
SELECT array_agg(name) FROM (
    SELECT *
    FROM nation
    ORDER BY name DESC
)
-- 改写为
SELECT array_agg(name ORDER BY name DESC) 
FROM nation
```

```sql
SELECT *, row_number() OVER ()
FROM (
    SELECT *
    FROM nation
    ORDER BY name DESC
)
-- 改写为
SELECT *, row_number() OVER (ORDER BY name DESC)
FROM nation
```

> 如果有的查询不想开启优化。在Trino中可以通过关闭优化器全局配置选项optimizer.skip-redundant-sort 或基于每个会话通过会话属性skip_redundant_sort关闭。注意这些选项将在未来的版本中删除。

## 深入源码

介绍完Trino对冗余OrderBy的优化，我们来看看源码是如何实现的。

```java
// StatementAnalyzer.java
protected Scope visitQuery(Query node, Optional<Scope> scope)
{
    Scope withScope = analyzeWith(node, scope);
    Scope queryBodyScope = process(node.getQueryBody(), withScope);

    List<Expression> orderByExpressions = emptyList();
    if (node.getOrderBy().isPresent()) {
        orderByExpressions = analyzeOrderBy(node, getSortItemsFromOrderBy(node.getOrderBy()), queryBodyScope);

        if ((queryBodyScope.getOuterQueryParent().isPresent() || !isTopLevel) && node.getLimit().isEmpty() && node.getOffset().isEmpty()) {
            // not the root scope and ORDER BY is ineffective
            // 不是最外层的查询且当前层没有limit或offset存在，标识orderby是冗余的
            analysis.markRedundantOrderBy(node.getOrderBy().get());
            warningCollector.add(new TrinoWarning(REDUNDANT_ORDER_BY, "ORDER BY in subquery may have no effect"));
        }
    }
    ... ...
}
```

analysis 会在构建查询计划时用到(关于执行计划生成参考[Trino源码学习-执行计划生成](/posts/9fa3672f/))。

```java
// QueryPlanner.plan
// -> QueryPlanner.orderingScheme
private Optional<OrderingScheme> orderingScheme(PlanBuilder subPlan, Optional<OrderBy> orderBy, List<Expression> orderByExpressions)
{
    // 如果开启跳过冗余Order By 且当前Order By被标记为冗余
    // 不构建执行计划SortNode
    if (orderBy.isEmpty() 
        || (isSkipRedundantSort(session) 
            && analysis.isOrderByRedundant(orderBy.get()))) {
        return Optional.empty();
    }
    ... ...
}
```


## 参考

- [1] [Trino.redundant-order-by](https://trino.io/blog/2019/06/03/redundant-order-by.html)
- [2] [Trino Issue.ORDER BY ignored when query parenthesized](https://github.com/trinodb/trino/issues/6008)
- [3] [Trino.Do not plan unnecessary Sort](https://github.com/trinodb/trino/pull/818)

