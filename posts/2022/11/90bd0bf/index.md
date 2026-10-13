# Clickhouse别名使用问题

clickhouse 在查询中使用别名时可能会有下面的问题[<sup>1</sup>](#refer-anchor-1)。

```sql
$ SELECT avg(number) AS number, max(number) FROM numbers(10)

Aggregate function avg(number) is found inside another aggregate function in query.
```

<!--more-->

> If aliased expression contains aggregate function, alias should not be resolved inside another aggregate function.  
> If alias name clashes with the column name, the substitution of this alias should be cancelled.

原因是clickhouse的别名如果和某个列名相同，就会有上面的异常。可以通过添加下面了配置处理上述问题[<sup>2</sup>](#refer-anchor-2).

```sql
SELECT avg(number) AS number, max(number) FROM numbers(10)
settings prefer_column_name_to_alias  = 1;
```

但是这个配置引入了新的问题[<sup>3</sup>](#refer-anchor-3).


> Prefer alias for ORDER BY after GROUP BY in case of set prefer_column_name_to_alias=1;  
> Clickhouse version 21.6  

```sql
$ SELECT
    max(number) AS number,
    sum(number) AS sum
FROM numbers(100)
ORDER BY number ASC

Query id: 058b2cb1-7ea6-44de-b6a8-662ac4d949a8


0 rows in set. Elapsed: 0.002 sec.

Received exception from server (version 21.6.1):
Code: 184. DB::Exception: Received from localhost:9000. DB::Exception: Aggregate function max(number) is found inside another aggregate function in query: While processing max(number) AS number.

$ set prefer_column_name_to_alias=1;

$ SELECT
    max(number) AS number,
    sum(number) AS sum
FROM numbers(100)
ORDER BY number ASC

Query id: f8ca2417-9bba-42c6-9f12-42987a26e75d


0 rows in set. Elapsed: 0.004 sec.

Received exception from server (version 21.6.1):
Code: 215. DB::Exception: Received from localhost:9000. DB::Exception: Column `number` is not under aggregate function and not in GROUP BY: While processing number ASC.
```

clickhouse 社区又提出了新的pr[<sup>4</sup>](#refer-anchor-4)处理上述问题:

> Add more options to prefer_column_name_to_alias setting. This is for #24237  
> 1. set prefer_column_name_to_alias=1 --> prefer column over alias in all sub clauses;  
> 2. set prefer_column_name_to_alias=2 --> prefer column over alias in all sub clauses before and include group by;  
> 3. set prefer_column_name_to_alias=3 --> prefer column over alias in all sub clauses before group by;

## 参考

<div id="refer-anchor-1"></div>

- [1] [Clickhouse.issue_9715](https://github.com/ClickHouse/ClickHouse/issues/9715)

<div id="refer-anchor-2"></div>

- [2] [Clickhouse.pull_22044](https://github.com/ClickHouse/ClickHouse/pull/22044)

<div id="refer-anchor-3"></div>

- [3] [Clickhouse.issue_24237](https://github.com/ClickHouse/ClickHouse/issues/24237)

<div id="refer-anchor-4"></div>

- [4] [Clickhouse.pull_24391](https://github.com/ClickHouse/ClickHouse/pull/24391)

