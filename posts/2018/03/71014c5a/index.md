# mysql索引

mysql的索引可以分为几类：

* 聚集索引和非聚集索引
* 唯一索引和普通索引
* 单列索引和组合索引
* 全文索引\(仅MyISAM支持\)
* 空间索引\(仅MyISAM支持\)
* ...

<!--more-->

## 聚集索引和非聚集索引

### 聚集索引\(InnoDB\)

按照每张表的主键构造一颗【顺序】B+树，同时叶子节点中存放的即为整张表的行记录数据，也将聚集索引的叶子节点称为数据页。聚集索引的这个特性决定了索引组织表中数据也是索引的一部分。同B+树数据结构一样，每个数据页都通过一个双向链表来进行链接；由于实际的数据页只能按照一颗B+树进行排序，因此每张表只能拥有一个聚集索引。在多数情况下，查询优化器倾向于采用聚集索引。对我们的启示：

- 所以通过主键查找的速度是最快的，先扫描到索引主键，再扫描主键对应的列值，两次IO就可以搞定；
- 每张表都必须创建自增的主键，如果在创建表时没有显式地定义主键,则InnoDB存储引擎会按如下方式选择或创建主键:
  - 首先判断表中是否有非空的唯一索引,如果有,则该列即为主键.
  - 如果不符合上述条件,Innodb表中在没有默认主键的情况下会生成一个6byte空间的自动增长主键，可以用select _rowid from table来查询。当在表里设置了主键之后，_rowid就是对应主键，select _rowid from table查询的是对应的主键值.
- Sql优化中消除性能瓶颈的最好方式就是向主键靠拢，主键不行可以考虑其他重复率低的索引；

在使用InnoDB存储引擎时，如果没有特别的需要，***请永远使用一个与业务无关的自增字段作为主键***。InnoDB使用聚集索引，数据记录本身被存于主索引（一颗B+Tree）的叶子节点上。这就要求同一个叶子节点内（大小为一个内存页或磁盘页）的各条数据记录按主键顺序存放，因此每当有一条新的记录插入时，MySQL会根据其主键将其插入适当的节点和位置，如果页面达到装载因子（InnoDB默认为15/16），则开辟一个新的页（节点）。所以在使用innoDB表时要避免随机的（不连续且值的分布范围非常大）聚簇索引，特别是针对I/O密集型的应用。例如：从性能角度考虑，使用UUID的方案就会导致聚簇索引的插入变得完全随机。

如果表使用自增主键，那么每次插入新的记录，记录就会顺序添加到当前索引节点的后续位置，当一页写满，就会自动开辟一个新的页。这样就会形成一个紧凑的索引结构，近似顺序填满。由于每次插入时也不需要移动已有数据，因此效率很高，也不会增加很多开销在维护索引上。

如果使用非自增主键（如果身份证号或学号等），由于每次插入主键的值近似于随机，因此每次新纪录都要被插到现有索引页得中间某个位置。

此时MySQL不得不为了将新记录插到合适位置而移动数据，甚至目标页面可能已经被回写到磁盘上而从缓存中清掉，此时又要从磁盘上读回来，这增加了很多开销，同时频繁的移动、分页操作造成了大量的碎片，得到了不够紧凑的索引结构，后续不得不通过OPTIMIZE TABLE来重建表并优化填充页面。

> 注意,MyISAM的索引文件仅仅保存数据记录的地址。在MyISAM中，主索引和辅助索引（Secondary key）在结构上没有任何区别，只是主索引要求key是唯一的，而辅助索引的key可以重复。MyISAM的索引方式也叫做“非聚集”的，之所以这么称呼是为了与InnoDB的聚集索引区分。

### 非聚集索引

按照索引(非主键)【顺序】创建B+树，但此时的叶子节点并不包含行记录的全部数据。叶子节点除了包含键值以外，每个叶子节点中的索引行中还包含了一个书签(bookmark)。该书签用来告诉innodb存储引擎哪里可以找到与索引相对应的行数据。当通过辅助索引来寻找数据时，innodb存储引擎会遍历辅助索引并通过叶级别的指针获得指向主键索引的主键，然后再通过主键索引来找到一个完整的行记录。

## 主键设计原则 & 常用方案

* MySQL主键应当是对用户没有意义的。
* MySQL主键应该是单列的，以便提高连接和筛选操作的效率
* 永远也不要更新MySQL主键
* MySQL主键应当有计算机自动生成,不应包含动态变化的数据，如时间戳、创建时间列、修改时间列等。
* 关于主键的类型选择上最常见的争论是用整型还是字符型的问题，关于这个问题《高性能MySQL》一书中有明确论断：整数通常是标识列的最好选择，因为它很快且可以使用AUTO_INCREAMENT,如果可能，应该避免使用字符串类型作为标识列，因为很消耗空间，且通常比数字类型慢。如果是使用MyISAM，则就更不能用字符型，因为MyISAM默认会对字符型采用压缩引擎，从而导致查询变得非常慢。

### 自增ID

优点:

* 数据库自动编号，速度快，而且是增量增长，聚集型主键按顺序存放，对于检索非常有利。
* 数字型，占用空间小，易排序，在程序中传递方便。

缺点:

* 不支持水平分片架构，水平分片的设计当中，这种方法显然不能保证全局唯一。
* 表锁问题
  * 在MySQL5.1.22之前，InnoDB自增值是通过其本身的自增长计数器来获取值，该实现方式是通过表锁机制来完成的（AUTO-INC LOCKING）。锁不是在每次事务完成后释放，而是在完成对自增长值插入的SQL语句后释放，要等待其释放才能进行后续操作。比如说当表里有一个auto_increment字段的时候，innoDB会在内存里保存一个计数器用来记录auto_increment的值，当插入一个新行数据时，就会用一个表锁来锁住这个计数器，直到插入结束。如果大量的并发插入，表锁会引起SQL堵塞。
  * 在5.1.22之后，InnoDB为了解决自增主键锁表的问题，引入了参数innodb_autoinc_lock_mode：
    * 0: 通过表锁的方式进行，也就是所有类型的insert都用AUTO-inc locking（表锁机制）。
    * 1: 默认值，对于simple insert 自增长值的产生使用互斥量对内存中的计数器进行累加操作，对于bulk insert 则还是使用表锁的方式进行。
    * 2: 对所有的insert-like 自增长值的产生使用互斥量机制完成，性能最高，并发插入可能导致自增值不连续，可能会导致Statement 的 Replication 出现不一致，使用该模式，需要用 Row Replication的模式。
* 自增主键不连续,当插入10条记录的时候，因为AUTO_INCREMENT=16，所以下次再插入的时候，主键就会不连续。

  ```sql
  Create Table: CREATE TABLE `tmp_auto_inc` (
    `id` int(11) NOT NULL AUTO_INCREMENT,
    `talkid` int(11) DEFAULT NULL,
    PRIMARY KEY (`id`)
  ) ENGINE=InnoDB AUTO_INCREMENT=16 DEFAULT CHARSET=gbk
  1 row in set (0.00 sec)
  ``` 

### uuid

优点:

* 全局唯一性、安全性、可移植性。
* 能够保证独立性，程序可以在不同的数据库间迁移，效果不受影响。
* 保证生成的ID不仅是表独立的，而且是库独立的，在你切分数据库的时候尤为重要

缺点:

* 针对InnoDB引擎会徒增IO压力，InnoDB为聚集主键类型的引擎，数据会按照主键进行排序，由于UUID的无序性，InnoDB会产生巨大的IO压力。InnoDB主键索引和数据存储位置相关（簇类索引），uuid 主键可能会引起数据位置频繁变动，严重影响性能。
* UUID长度过长，一个UUID占用128个比特（16个字节）。主键索引KeyLength长度过大，而影响能够基于内存的索引记录数量，进而影响基于内存的索引命中率，而基于硬盘进行索引查询性能很差。严重影响数据库服务器整体的性能表现。

### 自定义序列表

所谓自定义序列表，就是在库中建一张用于生成序列的表来存储序列信息，序列生成的策略通过程序层面来实现。如下所示，构建一张序列表：

```sql
CREATE TABLE `sequence` (
    `name` varchar(50) NOT NULL,
    `id` bigint(20) unsigned NOT NULL DEFAULT '0',
    PRIMARY KEY (`name`)
) ENGINE=InnoDB;
```
> 注意区别，id字段不是自增的，也不是主键。在使用前，我们需要先插入一些初始化数据：

```sql
INSERT INTO `sequence` (`name`) VALUES ('users'), ('photos'), ('albums'), ('comments');
```

接下来，我们可以通过执行下面的SQL语句来获得新的照片ID：

```sql
UPDATE `sequence` SET `id` = LAST_INSERT_ID(`id` + 1) WHERE `name` = 'photos';
SELECT LAST_INSERT_ID();
```

我们执行了一个更新操作，将id字段增加1，并将增加后的值传递到LAST_INSERT_ID函数， 从而指定了LAST_INSERT_ID的返回值。

实际上，我们不一定需要预先指定序列的名字。如果我们现在需要一种新的序列，我们可以直接执行下面的SQL语句：

```sql
INSERT INTO `sequence` (`name`) VALUES('new_business') ON DUPLICATE KEY UPDATE `id` = LAST_INSERT_ID(`id` + 1);
SELECT LAST_INSERT_ID();
```

这种方案的问题在于序列生成的逻辑脱离了数据库层，由应用层负责，增加了开发复杂度。当然，其实可以用spring来解决这一问题，因为在spring JDBC中已经对这种序列生成逻辑进行了简单的封装。我们可以看一下spring的相关源代码：MySQLMaxValueIncrementer.

```mermaid 
classDiagram
  DataFieldMaxValueIncrementer<|..AbstractDataFieldMaxValueIncrementer
  &lt;&lt;interface>> DataFieldMaxValueIncrementer
  AbstractDataFieldMaxValueIncrementer<|--AbstractColumnMaxValueIncrementer
  AbstractColumnMaxValueIncrementer<|--MySQLMaxValueIncrementer
```

DataFieldMaxValueIncrementer接口，定义了三个方法nextIntValue(), nextLongValue(), nextStringValue(), 就是以不同类型获取发号器的下一个值。

AbstractDataFieldMaxValueIncrementer实现DataFieldMaxValueIncrementer接口，是一个抽象类,它定义了三个属性:
- DataSource dataSource 数据源
- String incrementerName  发号表表名
- int paddingLength = 0 返回字符串类型补零位数

然后定义了一个抽象方法getNextKey(),`protected abstract long getNextKey();`将接口中的三个方法归一化成这一个方法，子类只需实现返回long类型返回值即可。

AbstractColumnMaxValueIncrementer继承AbstractDataFieldMaxValueIncrementer，还是个抽象类，定义了两个属性：
- String columnName;  // 发号序列的列名
- int cacheSize = 1;  // 内存缓存的大小

MySQLMaxValueIncrementer, 其核心就是两句SQL或者说就是last_insert_id()函数的使用，理解了这两句SQL就理解了这个类： `update columName set columnName last_insert_id(columnName + getCacheSize()); select last_insert_id();`首先第一句SQL将数据库中的序列值更新成当前值+缓存大小的值，并将此值赋给last_insert_id(), 然后再将last_insert_id()的值取出到应用程序中，相当于一次性从数据库中取出了缓存大小个序列值。

last_insert_id()， 返回本连接（connection）下，上一次成功插入操作的自增列的值，若插入操作为多行插入，则会返回第一行成功插入的自增值；此函数是连接维度的，不同的客户端连接，在MYSQL服务器端维护着不同的值。若同一客户端连接断了，再次重建，值也会重置，新建立的连接是不存在“上一次成功插入”的；是连接维度的，不是表维度的，表A表B都有自增列，SELECT last_insert_id()就是上一次成功插入操作的自增值，跟表无关；last_insert_id(expr)，原返expr值，并使得下一次系统自增值产生前调用LAST_INSERT_ID()也返回expr。


**这个方案分布式多线程会不会有问题？** 答:**不会**

- 因为last_insert_id()是基于connection的，所以分布式式部署环境下，发号不会乱；
- 因为getNextKey()方法由synchronized关键字修饰，并且正常的数据库连接池都不会将一个connection在同一时间分配给不同的线程，所以多线程环境下，发号也不会乱；

```java
/**
 * 使用MySQL表来实现的DataFieldMaxValueIncrementer自增发号器，与自增列（auto-increment）有同样的效果。
 * 注意: 如果你使用这个类, 你的MySQL表的主键可以不是自增的，由这个类来做这件事。
 *
 * 这个序列在一张表中维护，每个需要额外自增键的表需要这样一个序列表。表的存储引擎可以是MYISAM或INNODB，因为这个对这个序列的操作使用一个独立的连接，故不会受到其它事务的影响.
 *
 * 举例:
 *
 * create table tab (id int unsigned not null primary key, text varchar(100));
 * create table tab_sequence (value int not null);
 * insert into tab_sequence values(0);
 *
 * 如果设置了"cacheSize", 获取中间的值的时候可以避免访问数据库，但如果你的应用停止了或者服务器宕机了又或者事务回滚了，缓存里还没有使用的值就永远都不会被获取到并使用了，所以在序列中出现号码跳跃的最大区间大小为“cacheSize”的值。
 *
 * 可以通过设置“useNewConnection”的值为false来避免发号操作使用一个新的连接。但是这种情况下你得使用一个无事务的存储引擎，比如MYISAM。
 */
public class MySQLMaxValueIncrementer extends AbstractColumnMaxValueIncrementer {

	/** 获取新序列值的SQL */
    private static final String VALUE_SQL = "select last_insert_id()";
    /** 新的序列值 */
    private long nextId = 0;
    /** 当前内存缓存里可发序列的最大值 */
    private long maxId = 0;
	/** 本发号器每次发号是否使用一个新的数据库连接 */
	private boolean useNewConnection = true;

	/**
	 * 设置本发号器是否每次发号是否使用一个新的数据库连接
	 * 发号表使用支持事务的存储引擎则必须设置为true, 自增操作使用一个独立的事务；
	 * 发号表使用无事务的存储引擎（如MYISAM）设置为false就足够了,省去为自增发号操作获取一个新的连接
	 * 从Spring Framework 5.0.开始默认为false
	 * @since 4.3.6
	 */
	public void setUseNewConnection(boolean useNewConnection) {
		this.useNewConnection = useNewConnection;
	}

	@Override
	protected synchronized long getNextKey() throws DataAccessException {
		if (this.maxId == this.nextId) {
			/*
			* 如果useNewConnection为true, 获取一个未被管理的连接，让我们的操作在一个独立的事务中
			* 如果useNewConnection为false，使用当前事务的连接, 但依赖发号表是无事务的表.
            * 需要直接使用JDBC风格的代码，因为我们需要确保insert操作和select操作是在一个connection下进行的 (不然的话我们无法保证last_insert_id()返回正确的值)
			*/
			Connection con = null;
			Statement stmt = null;
			boolean mustRestoreAutoCommit = false;
			try {
				if (this.useNewConnection) {
                    // 若使用新连接，直接从数据源获取连接
					con = getDataSource().getConnection();
					if (con.getAutoCommit()) {
                        // 连接是自动提交的，则先设置为非自动提交
						mustRestoreAutoCommit = true;
						con.setAutoCommit(false);
					}
				} else {
					// 不使用新连接，尝试获取当前线程的连接
					con = DataSourceUtils.getConnection(getDataSource());
				}
				stmt = con.createStatement();
				if (!this.useNewConnection) {
                    // 设置事务语句超时时间
					DataSourceUtils.applyTransactionTimeout(stmt, getDataSource());
				}
				String columnName = getColumnName();
				try {
					// 更新库中的值为当前值+缓存大小，并将此值设置到last_insert_id()上
					stmt.executeUpdate("update " + getIncrementerName() + " set " + columnName +
							" = last_insert_id(" + columnName + " + " + getCacheSize() + ")");
				} catch (SQLException ex) {
					throw new DataAccessResourceFailureException("Could not increment " + columnName + " for " +
							getIncrementerName() + " sequence table", ex);
				}
				// 查询新的最大值（last_insert_id()）
				ResultSet rs = stmt.executeQuery(VALUE_SQL);
				try {
					if (!rs.next()) {
						throw new DataAccessResourceFailureException("last_insert_id() failed after executing an update");
					}
					// 获取新的缓存最大值
					this.maxId = rs.getLong(1);
				} finally {
					JdbcUtils.closeResultSet(rs);
				}
				this.nextId = this.maxId - getCacheSize() + 1;
			} catch (SQLException ex) {
				throw new DataAccessResourceFailureException("Could not obtain last_insert_id()", ex);
			} finally {
				JdbcUtils.closeStatement(stmt);
				if (con != null) {
					if (this.useNewConnection) {
						try {
                            // 提交事务
							con.commit();
							if (mustRestoreAutoCommit) {
								// 如果需要，恢复连接原设置
                                con.setAutoCommit(true);
							}
						} catch (SQLException ignore) {
							throw new DataAccessResourceFailureException(
									"Unable to commit new sequence value changes for " + getIncrementerName());
						}
						JdbcUtils.closeConnection(con);
					} else {
						DataSourceUtils.releaseConnection(con, getDataSource());
					}
				}
			}
		} else {
			this.nextId++;
		}
		return this.nextId;
	}

}
```

为什么要获取独立连接，免受其他事务影响? 

若自增操作getNextKey()调用前，本线程的连接就开启了一个事务，那么这个事务若回滚，数据库层面回滚了，业务数据（缓存最大值、当前值）却没有回滚，就会出现重复发号的问题。

spring的实现也就是通过update语句对incrementerName表里的columnName 列进行递增，并通过mysql的last_insert_id()返回最近生成的值。并保证了事务性及方法的并发支持。只是这个实现有些过于简单，比如：一个表对应一个序列的做法在实际应用开发中显得过于零碎，所以在实际应用中需要对其实现进行修改，实现一条记录对应一个序列的策略。另外对水平分片的支持并不在这一实现考虑范围内。同时，这种做法依然无法回避表锁的机制，所以这里通过CacheSize()的做法，实现了一次申请并缓存在内存中，以减少表锁的发生频率。

### 如何解决水平分片的需求

* UUID:由于UUID出现重复的概率基本可以忽略，所以对分片是天生支持的。
* 独立的序列库,单独建立一个库用来生成ID，在Shard中的每张表在这个ID库中都有一个对应的表，而这个对应的表只有一个字段， 这个字段是自增的。当我们需要插入新的数据，我们首先在ID库中的相应表中插入一条记录，以此得到一个新的ID， 然后将这个ID作为插入到Shard中的数据的主键。这个方法的缺点就是需要额外的插入操作，如果ID库变的很大， 性能也会随之降低。所以一定要保证ID库的数据集不要太大，一个办法是定期清理前面的记录
* 复合标识符,这种做法是通过联合主键的策略，即通过两个字段来生成一个唯一标识，前半部分是分片标识符，后半部分是本地生成的标识符（比如使用AUTO_INCREMENT生成）
* 带分库策略的自定义序列表,这种做法可以基于上面提到的自定义序列表的方法的基础上，做一些技巧性的调整。即如下：
  ```sql
  UPDATE `sequence` SET `id` = LAST_INSERT_ID(`id` + 1) WHERE `name` = 'photos';
  SELECT LAST_INSERT_ID();
  ```
  这里的id初始值设定上要求不同的分片取不同的值，且必须连续。同时将每次递增的步长设定为服务器数目。
  比如有3台机器，那么我们只要将初始值分别设置为1，2，3. 然后执行下面的语句即可：

  ```sql
  UPDATE `sequence` SET `id` = LAST_INSERT_ID(`id` + 3) WHERE `name` = 'photos';
  SELECT LAST_INSERT_ID();
  ```
  这就可以解决主键生成冲突的问题。但是如果在运行一段时间后要进行动态扩充分片数的时候，需要对序列初始值做一次调整，以确保其连续性，否则依然可能存在冲突的可能。当然这些逻辑可以封装在数据访问层的代码中。

## 唯一索引和普通索引

普通索引是mysql中的基本索引类型，允许在定义索引的列中插入空值和重复值。
唯一索引，索引的值必须唯一，允许有空值。主键索引是特殊的唯一索引，不允许有空值。

## 单列索引和组合索引

单列索引即一个索引只包含单个列，而组合索引指表的多个字段组合上创建索引。

## 索引的最左前缀原理

* 按照索引中所有列进行精确匹配（这里精确匹配指“=”或“IN”匹配）时，索引可以被用到。而在使用不等于(`!=`或者`<>`)的时候无法使用索引这里有一点需要注意，理论上索引对顺序是敏感的，但是由于MySQL的查询优化器会自动调整where子句的条件顺序以使用适合的索引，例如我们将where中的条件顺序颠倒,结果是一样的。
* 当查询条件精确匹配索引的左边连续一个或几个列时，又或者，查询条件用到了索引中列的精确匹配，但是中间某个条件未提供，索引可以被用到，但是只能用到一部分，即条件所组成的最左前缀。
* __查询条件没有指定索引第一列,这样的查询显然用不到索引__。
* 如果查询条件中含有函数或表达式，则MySQL不会为这列使用索引。
* 范围列可以用到索引（必须是最左前缀），但是范围列后面的列无法用到索引。同时，索引最多用于一个范围列，因此如果查询条件中有两个范围列则无法全用到索引。
* 匹配某列的前缀字符串,如果通配符%不出现在开头，则可以用到索引，但根据具体情况不同可能只会用其中一个前缀。例如，`SELECT * FROM employees.titles WHERE  title LIKE 'Senior%';`。

## 索引选择性与前缀索引

既然索引可以加快查询速度，那么是不是只要是查询语句需要，就建上索引？答案是否定的。因为索引虽然加快了查询速度，但索引也是有代价的：索引文件本身要消耗存储空间，同时索引会加重插入、删除和修改记录时的负担，另外，MySQL在运行时也要消耗资源维护索引，因此索引并不是越多越好。一般两种情况下不建议建索引。

第一种情况是表记录比较少，例如一两千条甚至只有几百条记录的表，没必要建索引，让查询做全表扫描就好了。至于多少条记录才算多，这个个人有个人的看法，我个人的经验是以2000作为分界线，记录数不超过 2000可以考虑不建索引，超过2000条可以酌情考虑索引。

另一种不建议建索引的情况是索引的选择性较低。所谓索引的选择性（Selectivity），是指不重复的索引值（也叫基数，Cardinality）与表记录数（#T）的比值：`Index Selectivity = Cardinality / #T`,显然选择性的取值范围为(0, 1]，选择性越高的索引价值越大，这是由B+Tree的性质决定的。

有一种与索引选择性有关的索引优化策略叫做前缀索引，就是用列的前缀代替整个列作为索引key，当前缀长度合适时，可以做到既使得前缀索引的选择性接近全列索引，同时因为索引key变短而减少了索引文件的大小和维护开销。

下面是一个选择长度的例子：

```sql
$ select count(distinct city) / count(*) from city_demo;
+---------------------------------+
| count(distinct city) / count(*) |
+---------------------------------+
|                          0.4283 |
+---------------------------------+

$ select count(distinct left(city,3))/count(*) as sel3,
-> count(distinct left(city,4))/count(*) as sel4,
-> count(distinct left(city,5))/count(*) as sel5, 
-> count(distinct left(city,6))/count(*) as sel6 
-> from city_demo;
+--------+--------+--------+--------+
| sel3   | sel4   | sel5   | sel6   |
+--------+--------+--------+--------+
| 0.3367 | 0.4075 | 0.4208 | 0.4267 |
+--------+--------+--------+--------+

-- 可以看见当索引前缀为6时的基数是0.4267，已经接近完整列选择性0.4283。
alter table city_demo add key (city(6));
```

