# MySQL参数 kill_idle_transaction

最近遇到一个问题，在执行长事务任务的过程中，频繁出现异常 `com.mysql.jdbc.exceptions.jdbc4.MySQLNonTransientConnectionException: Communications link failure during commit(). Transaction resolution unknown.`即，事务提交时发现连接已经失效。一开始以为是连接超时设置的有问题，但是这个异常重复出现，并且，数据库连接池设置了testOnBorrow。所以应该不是连接超时导致。后来发现，出现报错时，事务开启都刚好超过了5S。

经过和RDS同学的沟通。他们设置了kill_idle_transaction 这个参数，并且默认为5S

在线上遇到5.7.26的锁问题，需要解决idle事务长时间挂起的问题。同时也调研了现有的mysql timeout机制，以确保其和现有的timeout机制可以吻合。Percona从5.1.59-13.0引入了innodb_kill_idle_transaction，用于解决长事务场景，即对idle事务设定一个超时时间，对超过该时间的事务所在的用户连接进行断开。

<!--more-->

引入该参数也可以防止purge线程的长时间阻塞（长事务会一直保持在活跃状态，则会导致purge长时间的等待，从而导致undo无法清理从而造成磁盘空间的不断增加）。在实现上，开始是通过扫描InnoDB事务列表来进行判断的，在Percona Server 5.6.35-80.0则改为判断connection socket read timeout。这样优化的好处是，巡检可能会造成CPU空跑，而基于socket select超时则发生超时才会触发，使代码的运行更有效率。另外，percona现在提供了两个参数：innodb_kill_idle_transaction（后者的alias，5.7中已标记为deprecated）和kill_idle_transaction。我们在port时只保留kill_idle_transaction。

## kill_idle_transaction

MySQL官方版本不支持空闲事务自动kill的功能，Percona版本的MySQL引入了参数 kill_idle_transaction 和 innodb_kill_idle_transaction 来支持自动杀空闲事务。这两个参数本质上是一样的，innodb_kill_idle_transaction 是 kill_idle_transaction 参数的一个别名，在 5.7 版本，两个参数都可以使用，设置其中一个，另外一个也跟着变化。在 8.0 版本只保留了kill_idle_transaction参数。该参数设置的值为事务空闲的时间阈值，单位为秒，当一个事务空闲时间超过该阈值时，就会被强制杀掉。

kill_idle_transaction：

作用范围：全局，不支持session级修改
动态修改：是
默认值：0，即不杀空闲事务
看一个示例，设置kill_idle_transaction为5，然后开启一个事务，5秒之后，提交该事务，发现连接已经被杀掉，如下：

```sh
mysql> set global kill_idle_transaction=5;
Query OK, 0 rows affected (0.00 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> commit;
ERROR 2006 (HY000): MySQL server has gone away
No connection. Trying to reconnect...
Connection id:    309
Current database: db

Query OK, 0 rows affected (0.01 sec)
```

