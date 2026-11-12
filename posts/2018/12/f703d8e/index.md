# mysql使用问题:group concat

MySQL提供的`group_concat()`函数可以拼接某个字段值成字符串，如 `select group_concat(user_name) from sys_user`,默认的分隔符是逗号，即`,`，如果需要自定义分隔符可以使用 SEPARATOR.如：`select group_concat(user_name SEPARATOR '_')  from sys_user`。

但是如果 user_name 拼接的字符串的长度字节超过1024 则会被截断。通过命令`show variables like 'group_concat_max_len'`来查看group_concat 默认的长度：

```sql
mysql> show variables like 'group_concat_max_len';
+----------------------+-------+
| Variable_name | Value |
+----------------------+-------+
| group_concat_max_len | 1024 |
+----------------------+-------+
1 row in set
```

在MySQL配置文件中添加配置：group_concat_max_len = -1  （-1为最大值或根据实际需求设置长度），配置后需要重启MySQL服务，查看如下所示:

```sql
mysql> show variables like 'group_concat_max_len';
+----------------------+------------+
| Variable_name | Value |
+----------------------+------------+
| group_concat_max_len | 4294967295 |
+----------------------+------------+
1 row in set
```

如果是生产环境下，不能擅自重启MySQL服务，则可以通过语句设置group_concat的作用范围，如：

```sql
SET GLOBAL group_concat_max_len=-1;

SET SESSION group_concat_max_len=-1;
-- 使用mybatis时,可以在选择xml中设置 SESSION 中的 group_concat_max_len
```

