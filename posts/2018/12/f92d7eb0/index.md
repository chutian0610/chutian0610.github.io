# mysql报错 illegal mix of collations

尝试复现问题

* 首先创建数据库。

```sql
CREATE SCHEMA `tmp` DEFAULT CHARACTER SET utf8 COLLATE utf8_danish_ci ;
flush privileges;
grant all privileges on tmp.*  to 'victorchu'@'localhost' ;
```

* 接下来创建表A。

```sql
CREATE TABLE IF NOT EXISTS `tmp`.`A`(
  `id` INT NOT NULL,
  `name` VARCHAR(45) NULL,
  `b_name` VARCHAR(45) NULL,
  PRIMARY KEY (`id`))
ENGINE = InnoDB
DEFAULT CHARACTER SET = utf8;

INSERT INTO `tmp`.`A` (`id`, `name`, `b_name`) VALUES ('1', '1', '1');
INSERT INTO `tmp`.`A` (`id`, `name`, `b_name`) VALUES ('2', '2', '2');
INSERT INTO `tmp`.`A` (`id`, `name`, `b_name`) VALUES ('3', '3', '3');
```

接着创建表B。

```sql
CREATE TABLE IF NOT EXISTS `tmp`.`B`(
  `id` INT NOT NULL,
  `name` VARCHAR(45) NULL,
  `result` VARCHAR(45) NULL,
  PRIMARY KEY (`id`))
ENGINE = InnoDB;

INSERT INTO `tmp`.`B` (`id`, `name`, `result`) VALUES ('1', '1', '1');
INSERT INTO `tmp`.`B` (`id`, `name`, `result`) VALUES ('2', '2', '2');
INSERT INTO `tmp`.`B` (`id`, `name`, `result`) VALUES ('3', '3', '3');
```

异常查询。

```sql
use tmp;
select A.id , B.result
from A 
left join B on A.b_name = B.name
where A.id =1;

-- Error Code: 1267. Illegal mix of collations (utf8_general_ci,IMPLICIT) and (utf8_danish_ci,IMPLICIT) for operation '='	0.0011 sec
```

出现这个错的原因是相等的两个字段字符集不统一的问题。为了确认这一问题，我们来查看数据库字符集编码.

```sql
-- 查看MYSQL数据库服务器和数据库字符集
mysql> show variables like 'collation%';
+----------------------+--------------------+
| Variable_name        | Value              |
+----------------------+--------------------+
| collation_connection | utf8mb4_0900_ai_ci |
| collation_database   | utf8_danish_ci     |
| collation_server     | utf8mb4_0900_ai_ci |
+----------------------+--------------------+
-- 查看库的字符集
-- 结果省略部分
mysql> SHOW CREATE DATABASE tmp;
+----------+------------------------------------------------------------------------------+
| Database | Create Database                                                              |
+----------+------------------------------------------------------------------------------+
| tmp      | CREATE DATABASE `tmp` /*DEFAULT CHARACTER SET utf8 COLLATE utf8_danish_ci */ |
+----------+------------------------------------------------------------------------------+

-- 查看表的字符集
-- 结果省略部分列
mysql> show table status from tmp like 'A';
+-----------------+
| Collation       |
+-----------------+
| utf8_general_ci |
+-----------------+
mysql> show table status from tmp like 'B';
+----------------+
 Collation      |
+----------------+
| utf8_danish_ci |
+----------------+

-- 查看表中所有列的字符集
-- 结果省略部分列
mysql> show full columns from A;
+--------+-------------+-----------------+
| Field  | Type        | Collation       |
+--------+-------------+-----------------+
| id     | int         | NULL            | 
| name   | varchar(45) | utf8_general_ci | 
| b_name | varchar(45) | utf8_general_ci | 
+--------+-------------+-----------------+
mysql> show full columns from B;
+--------+-------------+----------------+
| Field  | Type        | Collation      | 
+--------+-------------+----------------+
| id     | int         | NULL           |
| name   | varchar(45) | utf8_danish_ci |
| result | varchar(45) | utf8_danish_ci | 
+--------+-------------+----------------+
```

## 解决方案

* 修改数据库字符集：`ALTER DATABASE db_name DEFAULT CHARACTER SET character_name [COLLATE ...];`
* 把表默认的字符集和所有字符列（CHAR,VARCHAR,TEXT）改为新的字符集：`ALTER TABLE tbl_name CONVERT TO CHARACTER SET character_name [COLLATE ...]`
* 修改表的默认字符集：`ALTER TABLE tbl_name DEFAULT CHARACTER SET character_name [COLLATE...];`
* 修改字段的字符集：`ALTER TABLE tbl_name CHANGE c_name c_name CHARACTER SET character_name [COLLATE ...];`

