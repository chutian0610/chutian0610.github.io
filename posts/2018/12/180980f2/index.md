# mysql-批量执行优化

本文介绍如何批量执行 mysql 语句。

<!--more-->

## allowMultiQueries

在 Mysql连接字符串上添加`allowMultiQueries=true`。可以支持在一个 Statement 中执行用分号隔开的多条 Sql 语句。例如下面的 mybatis语句

```xml
<update id="batchUpdate" parameterType="java.util.List">
  <foreach collection="list" item="item" separator=";" open="" close="">
    update test_table
    <set>
      <if test="item.a != null">output_amount = #{item.a},</if>
      <if test="item.b!= null">invoice_amount = #{item.b},</if>
      <if test="item.c!= null">payment_amount = #{item.c},</if>
    </set>
    where id = #{item.id}
  </foreach>
</update>
```

- 可以减少数据库连接次数，执行速度会更快
- 注意，执行多条的sql语句，但是只能返回一个结果集

## rewriteBatchedStatements

JDBC Statement 接口提供了addBatch方法，我们可以用它来批处理多个语句，以便在调用executeBatch 方法时在一个请求中发送这些语句，如下例所示。

```java
String INSERT = "insert into post (id, title) values (%1$d, 'Post no. %1$d')";

try(Statement statement = connection.createStatement()) {
    for (long id = 1; id <= 10; id++) {
        statement.addBatch(
            String.format(INSERT, id)
        );
    }
    statement.executeBatch();
}
```

上面的代码块不一定会在一次数据库往返中执行INSERT语句，可以从 mysql jdbc 的实现中确认。

```java
if (this.rewriteBatchedStatements.getValue() && nbrCommands > 4) {
    return executeBatchUsingMultiQueries(
        multiQueriesEnabled, 
        nbrCommands, 
        individualStatementTimeout
    );
}

updateCounts = new long[nbrCommands];

for (int i = 0; i < nbrCommands; i++) {
    updateCounts[i] = -3;
}

int commandIndex = 0;

for (commandIndex = 0; commandIndex < nbrCommands; commandIndex++) {
    try {
        String sql = (String) batchedArgs.get(commandIndex);
        updateCounts[commandIndex] = executeUpdateInternal(sql, true, true);
        
        ...
    } catch (SQLException ex) {
        updateCounts[commandIndex] = EXECUTE_FAILED;

        ...
    }
}
```

当rewriteBatchedStatements是 false 时，每个INSERT语句将使用executeUpdateInternal 方法调用单独执行。因此，我们还需要修改rewriteBatchedStatements属性，在 Jdbc url 上增加`&rewriteBatchedStatements=true`。

```java
// executeBatchUsingMultiQueries
StringBuilder queryBuf = new StringBuilder();

batchStmt = locallyScopedConn.createStatement();
JdbcStatement jdbcBatchedStmt = (JdbcStatement) batchStmt;

...

int argumentSetsInBatchSoFar = 0;

for (commandIndex = 0; commandIndex < nbrCommands; commandIndex++) {
    String nextQuery = (String) this.query.getBatchedArgs().get(commandIndex);

    ...

    queryBuf.append(nextQuery);
    queryBuf.append(";");
    argumentSetsInBatchSoFar++;
}

if (queryBuf.length() > 0) {
    try {
        batchStmt.execute(queryBuf.toString(), java.sql.Statement.RETURN_GENERATED_KEYS);
    } catch (SQLException ex) {
        sqlEx = handleExceptionForBatch(
            commandIndex - 1, argumentSetsInBatchSoFar, updateCounts, ex
        );
    }

    ...
}
```

根据`executeBatchUsingMultiQueries`的实现，可以发现对于普通的JDBCStatement 批处理，MySQL会使用`;`依次拼接 Statement并在单个数据库往返中执行它们。

但是对于PreparedStatement,MySQL JDBC驱动程序使用`executeBatchWithMultiValuesClause`方法，将成批的INSERT语句改写为单个多值INSERT。

```java
if (!this.batchHasPlainStatements && 
	this.rewriteBatchedStatements.getValue()) {

	if (getQueryInfo().isRewritableWithMultiValuesClause()) {
		return executeBatchWithMultiValuesClause(batchTimeout);
	}

	...
}
```

## 参考

- [1] [Mysql.Performance Extensions](https://dev.mysql.com/doc/connectors/en/connector-j-connp-props-performance-extensions.html)

