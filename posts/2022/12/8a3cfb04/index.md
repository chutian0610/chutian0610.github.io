# 分布式锁实现-DB

在[前面](posts/2020/05/4eb3381c/)我们简单介绍过分布式锁，本篇文章会介绍如果基于数据库(Mysql)实现分布式锁。基于Mysql实现锁的一般场景是对性能要求不高，且不希望因为分布式锁而引入新的组件的。

<!--more-->

## 乐观锁

乐观锁是分布式锁的常见实现。一种简单的实现是直接创建一张锁表，然后通过操作该表中的数据来实现加锁和解锁。

```SQL
CREATE TABLE `lock_table` (
    `id` INT(11) NOT NULL AUTO_INCREMENT COMMENT '自增id',
    `lock_name` VARCHAR(64) NOT NULL COMMENT '锁名',
    `lock_state` INT(11) NOT NULL COMMENT '锁状态',
    `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `uniq_method_name` (`lock_name`)
)
COMMENT='分布式锁'
COLLATE='utf8_general_ci'
ENGINE=InnoDB;
```

此时我们可以简单的使用insert/delete来快速加锁，解锁。

```SQL
insert into lock_table(lock_name, lock_state) values("methodName", "desc");
delete from lock_table where lock_name = "methodName";
```

当然，也可以使用Update方式加解锁，此时需要在进行分布式竞争前，向数据库插入一条记录，表示锁。

当要锁住某个方法或资源时，就向该表更新一条记录，表中设置方法名为唯一键，**这样多个请求同时提交数据库时，只有一个操作可以成功，判定操作成功的线程(更新数量>0)获得锁**，可以继续执行；

```sql
update lock_table set lock_state =1 where lock_name="lockName" and lock_state =0;
```

想要释放锁的时候就更新这条记录，其他线程就可以通过继续往更新数据获取锁。

```sql
update lock_table set lock_state =0 where lock_name="lockName" and lock_state =1;
```

> 注意，上述操作时，要主动commit，同时关闭spring的事务传播(`@Transactional(propagation = Propagation.NOT_SUPPORTED)`),否则我们的加锁事务在会等待外部事务一起提交，在默认的隔离级别下面，其他线程的事务是没办法读取未提交的事务，也就是说我们加的锁数据没有保存进数据库，其他线程一样可以加锁，这样就导致加锁失败了。

### 问题&应对方案

1.  锁强依赖数据库的可用性，数据库是一个单点，一旦数据库挂掉，会导致业务系统不可用；

- 两个数据库，数据库之间双向同步。一旦挂掉快速切换到备库上。

2. 锁没有失效时间，一旦解锁操作失败，就会导致锁记录一直在数据库中，其他线程无法再获得到锁；

- 在数据库表中加个字段，记录每把锁的过期时间，再做一个定时任务，定期清理数据库中的超时数据；

3. 锁只能是非阻塞的，因为数据操作，一旦插入失败就会直接报错。没有获得锁的线程并不会进入排队队列，要想再次获得锁就要再次触发获得锁操作；

- 客户端自旋，直到加锁成功再返回；

4. 锁是非重入的，同一个线程在没有释放锁之前无法再次获得该锁。因为数据表中数据已经存在了。
 
- 在数据库表中加个字段，记录当前获得锁的机器的主机信息,线程信息和进入次数。


```sql
CREATE TABLE `lock_table` (
    `id` INT(11) NOT NULL AUTO_INCREMENT COMMENT '自增id',
    `expiration_time` datetime DEFAULT NULL COMMENT '过期时间',
    `lock_name` VARCHAR(64) NOT NULL COMMENT '锁名',
    `lock_state` INT(11) NOT NULL COMMENT '锁状态',
    `lock_info` VARCHAR(64) NOT NULL COMMENT '锁信息',
    `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (`id`),
    UNIQUE INDEX `uniq_method_name` (`lock_name`)
)
COMMENT='分布式锁'
COLLATE='utf8_general_ci'
ENGINE=InnoDB;

-- clientA
-- 第一次进入
select * from lock_table where lock_name="lockName";
update lock_table set lock_state =1 and lock_info = "clientA_thread1_1" and expiration_time=(now()+interval 1 min) where lock_name="lockName" and lock_state =0;
-- 第二次进入
select * from lock_table where lock_name="lockName";
update lock_table set lock_info = "clientA_thread1_2" and expiration_time=(now()+interval 1 min) where lock_name="lockName" and lock_state =1 and lock_info = "clientA_thread1_1";

--clientB
select * from lock_table where lock_name="lockName";
update lock_table set lock_state =1 and lock_info = "clientB_thread2_1" and expiration_time=(now()+interval 1 min) where lock_name="lockName" and lock_state =0;
-- update失败，自旋重试等待
```

> 拿数据不上锁。更新数据判断该期间是否被改过（版本号等机制），期间可读。适合修改少，读多，提高吞吐量，少量冲突，省大量锁的开销。经常冲突（写多）的情况下，上层不断retry，反而降低了性能，悲观锁更合适。

## 悲观锁

我们还可以通过数据库的排他锁来实现分布式锁，基于mysql的InnoDB引擎，使用如下方法来加锁：

```sql
--设置MySQL为非autocommit模式：
set autocommit=0;
--设置完autocommit后，我们就可以执行我们的正常业务了。具体如下：
--开始事务
start transaction; 
-- 获取锁，如果无法获取，会阻塞
select * from lock_table where lock_name="lockName" for update;

-- 进行一些业务操作
update lock_table set lock_state =1 where lock_name="lockName" and lock_state =0;

--提交事务，释放锁
commit;
```

在查询语句后面增加for update，数据库会在查询过程中给数据库表增加排他锁(这里再多提一句，InnoDB引擎在加锁的时候，只有通过索引进行检索的时候才会使用行级锁，否则会使用表级锁。这里我们希望使用行级锁，就要给lock_name添加索引，值得注意的是，这个索引一定要创建成唯一索引)。当某条记录被加上排他锁之后，其他线程无法再在该行记录上增加排他锁。
我们可以认为获得排它锁的线程即可获得分布式锁，当获取到锁之后，可以执行方法的业务逻辑，执行完方法之后，再通过Commit解锁。对应的代码如下。

```java
public void createByLock(Runnable runner) {
        Connection connection = null;
        Boolean connAutoCommit = null;
        PreparedStatement preparedStatement = null;
        try {
            connection = dataSource.getConnection();
            connAutoCommit = connection.getAutoCommit();
            //设置连接为手动提交
            connection.setAutoCommit(false);
            preparedStatement = connection.prepareStatement("select * from lock_table where lock_name='lockName' for update;");
            //获取锁 (如果当前事务没有提交别的线程进来会阻塞)
            preparedStatement.execute();
            //执行业务操作
            runner.run();
        } catch (Exception ex) {
            logger.error(ex.toString());
        } finally {
            if (connection != null) {
                try {
                    //提交事务
                    connection.commit();
                    //恢复自动提交
                    connection.setAutoCommit(connAutoCommit);
                    connection.close();
                } catch (SQLException ex) {
                    logger.error(ex.toString());
                }
            }
            if (preparedStatement != null) {
                try {
                    preparedStatement.close();
                } catch (SQLException ex) {
                    logger.error(ex.toString());
                }
            }
        }
    }
```

这种方法可以有效的解决乐观锁中提到的无法释放锁和阻塞锁的问题。 for update语句会在执行成功后立即返回，在执行失败时一直处于阻塞状态，直到成功。服务宕机之后数据库会自己把锁释放掉。但是还是无法直接解决数据库单点和可重入问题。

除此之外，还可能引入新的问题:

- 虽然我们使用了唯一索引，并且显示使用for update来使用行级锁。如果我们是使用插入方式申请锁(`insert into lock_table(lock_name, lock_desc) values("methodName", "desc");`)，那么`select for update`可能锁住的是GAP，如果此时另一个事务也获得同样的GAP锁，那么可能发生死锁。因此建议先创建锁资源，使用Update语句进行锁申请。
- 我们要使用排他锁来进行分布式锁的lock，那么一个排他锁长时间不提交，就会占用数据库连接。一旦类似的连接变得多了，就可能把数据库连接池撑爆。

### Select For Update 死锁的说明


下面通过一个比较简单的例子复现一下这个场景
首先给表里初始化3条数据。

```sql
insert into number select 'bbb',2;
insert into number select 'hhh',8;
insert into number select 'yyy',25
```

接着按照如下的时序进行操作：

|session 1|session 2|说明|
|:---|:---|:---|
|begin;||
||begin;|
|select * from number where prefix='ddd' for update;||session1事务拿到了hhh前的gap锁，也就是('bbb', 'hhh')的gap锁|
||select * from number where prefix='fff' for update|session2事务也拿到了hhh前的gap锁|
|insert into number select 'ddd',1||session 1 insert 获取插入意向锁，等待session 2的gap锁释放|
|锁等待中|insert into number select 'fff',1|session 2 insert 获取插入意向锁，等待session 1的gap锁释放|
|锁等待解除|死锁，session2的事务被回滚|循环等待，死锁检测，Innodb回滚session2事务|


这种情况出现的原因是：两个session同时通过select for update，并且未命中任何记录的情况下，是有可能得到相同gap的锁的（要看where筛选条件是否落在同一个区间。如果上面的案例如果一个session准备插入'ddd'另一个准备插入'kkk'则不会出现冲突，因为不是同一个gap）。此时再进行并发插入，其中一个会进入锁等待，待第二个session进行插入时，会出现死锁。MySQL会根据事务权重选择一个事务进行回滚。

那么如何避免这个情况呢？一种解决办法是将事务隔离级别降低到Read Committed，这时不会有gap锁，对于上述场景，如果where中条件不同即最终要插入的键不同，则不会有问题。如果业务代码中可能不同线程会尝试对相同键进行select for update，则可在业务代码中捕获索引冲突异常进行重试。

此外，上面代码示例中的代码还有一处值得注意的地方是事务注解@Transactional的传播机制，对于这类与主流程事务关系不大的方法，应当将事务传播行为改为REQUIRES_NEW。原因有两点:

- 因为这里的解决方案是对隔离级别降级，如果传播行为仍然是默认的话，在外层事务隔离级别不是RC的情况下，会抛出IllegalTransactionStateException异常(在你的TransactionManager开启了validateExistingTransaction校验的情况下)。
- 如果加入外层事务的话，某个线程在执行获取流水号的时候可能会因为另一个线程的与流水号不相关的事务代码还没执行完毕而阻塞。

