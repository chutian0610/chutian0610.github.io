# HikariCP Connection Adder 卡死：JDBC URL 漏配 `socketTimeout` 的故障复盘


先看一段栈，再决定要不要继续读——

```bash
# 这两份 stack dump 间隔 14 分钟抓的，"jdbc:mysql://10.14.127.31:19030/xxx"
# 的 connection adder 线程，tid 完全一致，栈帧一帧没动。
# 业务侧呢？告警群里全是 SQLTransientConnectionException。
```

说真的，这种栈谁都不想看到——10 个 HikariCP 连接池、每个池的 adder 都卡在 `socketRead0`，不是忙，是死等。更扎心的是，业务方第一时间甩锅 HikariCP "卡死了"，但 HikariCP 的 `MAIN` 池好好的；另一边 DBA 看到 MySQL 进程没崩、`SHOW PROCESSLIST` 也查不出明显异常。两边都没事，问题到底出在哪？

排查过程中我们发现了一个让人有点尴尬的真相：**故障既不在 HikariCP 框架本身，也不在 MySQL 服务端，而是 JDBC URL 上漏写了一个参数——`socketTimeout`**。一行配置就能解决，不需要升级任何东西。这篇文章会顺着这个发现，讲清楚这次故障是怎么发生的、为什么很多"看起来合理"的调参救不了命，以及怎么用一个脚本批量检查所有 JDBC URL。

如果你的 JDBC URL 上**没显式配 `socketTimeout`**，**强烈建议你先跳到 §6 把 Q7 与 Q7 补充 读完**——它们会用 `MysqlIO` 构造器的字节码告诉你，握手 read 默认没有任何超时保护，以及一行 `socketTimeout=20000` 是怎么把这件事修掉的。

## 0. 一页纸结论

- **现象**：应用 10 个 HikariCP 连接池同时失效，业务侧持续报 `SQLTransientConnectionException: Connection is not available, request timed out`。
- **直接卡点**：每个池的 `connection adder` 线程均卡在 `socketRead0`，两份 jstack 间隔 14 分钟、同一组 tid、同一帧栈。
- **根因**：**JDBC URL 上没配 `socketTimeout`**——连接池的 `connectionTimeout` 等参数管不到 JDBC driver 的 socket read；驱动只在显式配了 `socketTimeout` 时才会在 `MysqlIO` 构造器里设 `SO_RCVTIMEO`，否则握手 read 永远无超时保护。这是**客户端配置问题**，和 mysql / 驱动 / JDK 版本无关。
- **修复**：在 JDBC URL 上加 `socketTimeout=20000&connectTimeout=3000`——一行配置就能解决，不用升级任何东西。
- **预防**：所有生产环境的 MySQL JDBC URL 都必须显式设 `socketTimeout` 与 `connectTimeout`；连接池的超时参数和 driver 的 socket 超时是两套独立的机制，必须配合使用。

## 1. 故障现场

### 1.1 时间线

| 时间 | 事件 |
|---|---|
| T0 | 10 个业务连接池（g_caiwu_stream / soda_*_stream / ...）陆续报 `Connection is not available, request timed out after 30000ms` |
| T0+7min | 第一次 jstack（stack.log），10 个 adder 全部阻塞 |
| T0+21min | 第二次 jstack（stack1.log），与第一次同一组 tid、同一帧栈 |
| T0+30min | 重启客户端服务——重启让现有连接被释放、adder 短暂能建连接，告警暂时消失 |
| T0+~1h | 同一故障再次复发（因为 JDBC URL 没改） |

### 1.2 影响面

- 受影响池：10 个（同一 MySQL 端点 `10.14.127.31:19030`）
- 未受影响池：`MAIN_HikariCP`——说明问题在业务数据源侧，不在 HikariCP 框架本身
- 影响范围与版本无关：故障在 JDK 8 / 11 / 17 / 21 都同样会出现，5.1.x / 8.x 驱动也都会出现——**根因是 JDBC URL 漏配 `socketTimeout`**

### 1.3 关键 stack（pool-A）

```
"jdbc:mysql://10.14.127.31:19030/pool-A?... connection adder" #7952751 daemon
   java.lang.Thread.State: RUNNABLE
       at java.net.SocketInputStream.socketRead0(Native Method)
       at java.net.SocketInputStream.socketRead(SocketInputStream.java:116)
       at java.net.SocketInputStream.read(SocketInputStream.java:171)
       at java.net.SocketInputStream.read(SocketInputStream.java:141)
       at com.mysql.jdbc.util.ReadAheadInputStream.fill(ReadAheadInputStream.java:101)
       at com.mysql.jdbc.util.ReadAheadInputStream.readFromUnderlyingStreamIfNecessary(ReadAheadInputStream.java:144)
       at com.mysql.jdbc.util.ReadAheadInputStream.read(ReadAheadInputStream.java:174)
       at com.mysql.jdbc.MysqlIO.readFully(MysqlIO.java:3011)
       at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3472)
       at com.mysql.jdbc.MysqlIO.reuseAndReadPacket(MysqlIO.java:3462)
       at com.mysql.jdbc.MysqlIO.checkErrorPacket(MysqlIO.java:3905)
       at com.mysql.jdbc.MysqlIO.sendCommand(MysqlIO.java:2530)
       at com.mysql.jdbc.MysqlIO.sqlQueryDirect(MysqlIO.java:2683)
       at com.mysql.jdbc.ConnectionImpl.execSQL(ConnectionImpl.java:2491)
       at com.mysql.jdbc.ConnectionImpl.loadServerVariables(ConnectionImpl.java:3797)   ← 卡点
       at com.mysql.jdbc.ConnectionImpl.initializePropsFromServer(ConnectionImpl.java:3230)
       at com.mysql.jdbc.ConnectionImpl.connectOneTryOnly(ConnectionImpl.java:2243)
       at com.mysql.jdbc.ConnectionImpl.createNewIO(ConnectionImpl.java:2025)
       at com.mysql.jdbc.ConnectionImpl.<init>(ConnectionImpl.java:778)
       at com.mysql.jdbc.JDBC4Connection.<init>(JDBC4Connection.java:47)
       ... 反射构造 ...
       at com.mysql.jdbc.NonRegisteringDriver.connect(NonRegisteringDriver.java:330)
       at com.zaxxer.hikari.util.DriverDataSource.getConnection(DriverDataSource.java:138)
       at com.zaxxer.hikari.pool.PoolBase.newConnection(PoolBase.java:364)
       at com.zaxxer.hikari.pool.PoolBase.newPoolEntry(PoolBase.java:206)
       at com.zaxxer.hikari.pool.HikariPool.createPoolEntry(HikariPool.java:476)
       at com.zaxxer.hikari.pool.HikariPool$PoolEntryCreator.call(HikariPool.java:726)
```

少数 pool 走的是另一条握手路径 `setupServerForTruncationChecks`（被 `jdbcCompliantTruncation=true` 触发），栈底完全一致。

### 1.4 两次 dump 的交叉验证

两份 stack dump 间隔 14 分钟，10 个卡住的线程 tid / nid / 栈帧完全一致。这是判断"阻塞"而非"繁忙"的关键证据：

- `RUNNABLE` + `socketRead0` 不代表正在执行，仅说明 OS 调度层面可运行。
- 两次 dump 间隔 14 分钟栈帧一致，且多个池同步发生——可以判定为阻塞在 syscall。

## 2. 根因分析

### 2.1 两层归因

```
┌──────────────────────────────────────────────────────────────────┐
│ 应用层：HikariCP 连接池没配 socketTimeout，所有连接池同步失效        │
├──────────────────────────────────────────────────────────────────┤
│ 驱动层：默认 URL 下 mysql-connector-java 不设 SO_RCVTIMEO            │
└──────────────────────────────────────────────────────────────────┘
```

注：本次故障里服务端不响应只是触发条件，不是根因。如果客户端配了 socketTimeout，即使服务端出问题，握手 read 也会在超时后抛出 `SocketTimeoutException`，HikariCP adder 会失败但不会卡死。

### 2.2 为什么是客户端配置缺失

握手阶段死等的调用栈：

```
MysqlIO.readFully
  → ReadAheadInputStream.fill
  → SocketInputStream.read
  → socketRead0 (native, 阻塞)
```

**最关键的机制**：JDBC URL 上 `socketTimeout` 参数的默认值是 0，而 mysql-connector-java 的 `MysqlIO` 构造器在 TCP 连接建立之后只会做下面这件事：

```text
474: iload         6               ← 加载 socketTimeout 参数
476: ifeq          493              ← socketTimeout == 0 → 跳过
480: getfield      mysqlConnection:Socket
483: iload         6
485: invokevirtual Socket.setSoTimeout(I)V
488: goto          493
```

也就是说 **MysqlIO 构造器在 `socketTimeout == 0` 时不调 setSoTimeout**，socket 上保留 JDK 默认的 `SO_RCVTIMEO = 0`（即无超时）。一旦 socket 上没设超时，`socketRead0` 就只能等服务端主动关连接才会返回——对端不响应就永久阻塞。

参数与生效的对照表：

| 配置 | 默认值 | 在握手 read 上是否生效 | 原因 |
|---|---|---|---|
| JDBC URL `connectTimeout` | 0（典型） | 控制 TCP 三次握手 | 卡点不在 connect |
| JDBC URL `socketTimeout` | **0** | **默认不生效** | `MysqlIO` 构造器 `if (socketTimeout != 0)` 守卫跳过 setSoTimeout |
| `Statement.setQueryTimeout` | 0 | 无效 | 连接尚未建立 |
| `tcpKeepAlive=true` | 5.1.47 默认 false | 默认不生效 | 5.1.49 之前默认 false；Linux 默认 `tcp_keepalive_time=7200` |

**如果在 URL 里显式设了 `socketTimeout=20000`，握手 read 是会被中断的**——一行配置就能解决这个问题，不需要升级任何东西。

### 2.3 连接池的超时参数为什么救不了

| 连接池配置 | 是否能中断握手 read | 原因 |
|---|---|---|
| HikariCP `connectionTimeout` | 否 | 控制业务 `getConnection()` 的等待时长，不涉及驱动建连过程 |
| HikariCP `validationTimeout` | 否 | 仅用于探测已有连接 |
| HikariCP `keepaliveTime` / `maxLifetime` | 否 | 用于驱逐存量连接 |
| HikariCP `initializationFailTimeout` | 否 | 仅在启动期生效 |
| HikariCP 4.x `async-init` | 部分缓解 | 仅将"握手后"逻辑切出主线程，握手 read 仍在 `newConnection()` 内，adder 仍同步阻塞 |

**关键认识**：连接池的超时参数（无论是 HikariCP 还是 Druid）和 JDBC driver 的 socket 超时，是**两套独立的机制**——前者管"池"层，后者管"协议"层。配置 HikariCP 的任何超时参数都救不了握手 read。

关键结论：**任何 Java 层超时都无法中断 native `socketRead0`**——线程阻塞在 syscall，Java 层没有机会检查中断标志。

### 2.4 为什么 10 个池同时失效

这 10 个池全部指向同一 MySQL 端点 `10.14.127.31:19030`，并且**全部都没配 socketTimeout**。故障链大致是这样的：

1. 池里现连接被服务端踢除（例如 `wait_timeout` 过期、KILL 连接），或连接池因为 `maxLifetime` 主动驱逐。
2. 池瞬间需要批量重建连接，多个池的 adder 同时向 MySQL 发起握手。
3. 某个服务端诱因在同一时刻被踩中——`max_connections` 打满、主从切换、VIP 漂移、磁盘 IO 抖动等，导致新握手在服务端被排队或丢包。
4. 由于 socket 上没有 `SO_RCVTIMEO`，所有 adder 上的握手 read 全部死等，10 个池同步卡死。

服务端常见诱因（按出现概率从高到低）：

1. `max_connections` 打满：新握手被排队，客户端 read 死等。
2. DNS 反向解析慢：`gethostbyaddr` 使服务端卡在 login 阶段。
3. 主从切换 / VIP 漂移：包被发往旧主后丢弃，客户端 read 永不 ACK。
4. 磁盘 IO 抖动：MySQL 握手线程被 IO 调度阻塞。
5. LB / 防火墙会话表过期：握手包过去、回包被丢弃。

> 再次强调：服务端这些诱因都是"触发条件"，不是根因。如果配了 socketTimeout，这些诱因发生时 HikariCP 会快速抛 `SQLTransientConnectionException`（带"连接获取超时"原因），业务方第一时间能看到，而不是看到这种"卡死 10 个池 14 分钟栈帧一动不动"的诡异故障。

### 2.5 与 HikariCP#2161 的关系

| 维度 | #2161 | 本次故障 |
|---|---|---|
| 表象 | 同类：adder 卡驱动握手 socket read | 同类 |
| 阻塞阶段 | Oracle T4C logon | MySQL `loadServerVariables` / `setupServerForTruncationChecks` |
| **根因** | **Oracle JDBC URL 没配 socketTimeout** | **MySQL JDBC URL 没配 socketTimeout** |
| **修复** | **Oracle URL 加 socketTimeout** | **MySQL URL 加 socketTimeout** |

注：#2161 与本次故障的根因**完全同类**——都是"JDBC URL 没配 socketTimeout"。两份问题的修复方法也是**完全同类**——都是在 JDBC URL 上显式配 socketTimeout。两个驱动表现一致，证据结构也一致。

## 3. 排查手册

故障定位的核心思路就一句话：**先排除"是不是卡在驱动 socket read 上"，再决定下一步是改客户端还是找 DBA**。下面这一节是按这个思路写的可复用 SOP。

### 3.1 一分钟识别

判断"HikariCP 连接池被打挂且是 adder 卡驱动"的三项特征：

- [ ] stack dump 中 `XxxDataSource connection adder` 线程处于 `RUNNABLE`，但栈底位于 `socketRead0` / `read0`
- [ ] 多次 dump（间隔 ≥5 分钟）这些 adder 的 tid、栈帧完全一致
- [ ] HikariPool housekeeper 报 `Pool stats (total=0, active=0, idle=0, waiting=N)` 并伴有 `Add connection elided` 日志

任一满足即可按下文继续排查。

### 3.2 信息收集 checklist

应用侧（RD / 中间件）：

- 依赖版本：HikariCP / mysql-connector-java / JDK
- HikariCP 配置：`maximumPoolSize` / `minimumIdle` / `connectionTimeout` / `validationTimeout` / `keepaliveTime` / `maxLifetime` / `initializationFailTimeout`
- **JDBC URL 完整参数**（重点关注 `useServerPrepStmts` / `cachePrepStmts` / `jdbcCompliantTruncation` / `connectTimeout` / `socketTimeout` / `tcpKeepAlive`）——**特别是有没有显式设 `socketTimeout`**
- 故障窗口两次 jstack，间隔 ≥10 分钟（用于对比 tid / 栈帧是否一致）
- HikariPoolMXBean 的 active / idle / waiting / total 时间线
- `SQLTransientConnectionException` 首次出现的时间戳
- 可用 `parse_hikari_log_v2.py` 处理 `monitor.log`，提取 `Add connection elided` / `Pool stats` 时间线

DBA / 服务端侧：

- 故障 MySQL 的 `max_connections` / `Threads_connected` / `Threads_running` 峰值
- `SHOW PROCESSLIST` 中是否存在 `unauthenticated user | login` 堆积
- MySQL error log：是否存在连接拒绝、OOM、KILL 事件
- 主从状态：是否存在切换、relay log 损坏
- 磁盘 IO：故障窗口的 `iostat` / `iotop`
- 网络：是否存在 VIP 漂移、交换机异常、防火墙事件
- `skip-name-resolve` 是否已开启

### 3.3 三步定位法

**Step 1. 客户端抓包** — 30 秒内可区分服务端与驱动的责任边界

```bash
tcpdump -i any -nn -s 0 host 10.14.127.31 and port 19030 -w /tmp/dump.pcap
```

配合 Wireshark 查看握手 read 是否有回包：

- 有回包 → 驱动未正确解析（升级驱动）
- 无回包 → 服务端未响应（继续 Step 2）

**Step 2. 服务端确认** — 由 DBA 执行

```sql
-- 快速看连接是否被打满、是否有未完成握手
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Threads_running';
SHOW PROCESSLIST;                          -- 重点观察是否存在 unauthenticated | login 堆积
SHOW VARIABLES LIKE 'skip_name_resolve';   -- 未开启 = 握手阶段会做 DNS 反查，慢/挂会拖死连接

-- 故障窗口的 SQL 聚合（需 performance_schema 开启；<故障窗口> 替换为实际起止时间戳）
SELECT *
FROM performance_schema.events_statements_summary_by_digest
WHERE first_seen BETWEEN '2026-08-11 20:30:00' AND '2026-08-11 21:30:00'
ORDER BY count_star DESC
LIMIT 20;
```

**Step 3. 应用层止血** — 不依赖根因修复

**手动方案**（紧急止血）：在故障池的 HikariConfig 上调小 `connectionTimeout`（如 1 秒），让业务线程快速失败；或调用 `HikariDataSource.getHikariPoolMXBean().softEvictConnections()` 强制驱逐当前连接。

**自动化方案**见 §3.4 方案 D 的 watchdog 代码——按条件判断后再 evict，避免误伤健康池。

注意：若服务端仍无响应，重连后会再次死等——本步骤只能争取时间，不能替代根因修复（**根因修复见 §3.4 方案 A：URL 加 `socketTimeout`**）。

### 3.4 修复方案

#### 方案 A：在 JDBC URL 上加 `socketTimeout`（一行配置，根治）

这是**本次故障的根治方案**——一行 URL 参数就能解决，不需要升级任何东西。

```
jdbc:mysql://10.14.127.31:19030/db?...&connectTimeout=3000&socketTimeout=20000&tcpKeepAlive=true
```

每个参数的含义：

- `connectTimeout=3000`：TCP 三次握手超时 3 秒
- `socketTimeout=20000`：新建连接 + 业务 SQL 的 socket read 超时 20 秒——**关键参数**
- `tcpKeepAlive=true`：开启 TCP keepalive（注意：5.1.47 默认 false，5.1.49+ 默认 true）

为什么这一行就够了：实证见 §2.2 —— `MysqlIO` 构造器在 `socketTimeout != 0` 时会调 `setSoTimeout`，握手 read 会在 20 秒后抛 `SocketTimeoutException`，adder 会失败但不会卡死。

#### 方案 B：可选——升级 mysql-connector-java

如果你在生产环境长期依赖 mysql-connector-java 5.1.x，建议同步升级到 5.1.49（仅改 `tcpKeepAlive` 默认值）：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.49</version>
</dependency>
```

如果要彻底迁 8.x（包名 `com.mysql.cj.*`、协议层重写），注意：

- 包名 `com.mysql.jdbc.*` → `com.mysql.cj.*`
- 默认时区 / SSL 行为有变化
- 部分历史参数不再支持（如 `useOldUTF8Behavior`）
- 强烈建议灰度验证

但**注意**：URL 配置缺失的问题与驱动版本无关，**升 8.x 同样需要 URL 上显式配 `socketTimeout`**。所以 URL 配置是不可省略的。

#### 方案 C：HikariCP 调优（缓解，不能根治）

```yaml
hikari:
  connection-timeout: 5000          # 业务 getConnection 超时
  validation-timeout: 2000          # housekeeper 探测
  initialization-fail-timeout: 1    # 启动期快速失败
  keepalive-time: 30000             # 30s 心跳
  max-lifetime: 1800000             # 30min 驱逐
  minimum-idle: 5                   # 保持最低水位
```

提醒：这些参数都无法解决"新建连接握手死等"，它们和 driver 的 socket 超时是两套独立的机制。**唯一有效的是 §3.4 方案 A**。

#### 方案 D：应用层 watchdog（兜底防线）

```java
@Scheduled(fixedDelay = 30_000)
public void hikariWatchdog() {
    hikariPools.forEach(pool -> {
        var mx = pool.getHikariPoolMXBean();
        if (mx.getTotalConnections() == 0
                && mx.getIdleConnections() == 0
                && mx.getThreadsAwaitingConnection() > 0) {
            // 池已被打挂：驱逐卡死连接，触发重建
            mx.softEvictConnections();
            alert("Hikari pool " + pool.getPoolName() + " stuck, evicted");
        }
    });
}
```

软驱逐会触发下一次 fill 提交新 `FutureTask`；前提是服务端已恢复。

#### 方案 E：服务端修复（治本之策）

- 开启 `skip-name-resolve`
- 调高 `max_connections` / `back_log`，并合理设置 `wait_timeout` / `interactive_timeout`
- 主从切换接入 ProxySQL / MHA 自动重试
- 关键池接入 VIP 健康检查与漂移告警

## 4. 经验沉淀

### 4.1 关于 HikariCP 的几个常见误解

1. "`connectionTimeout` 能管所有阻塞"——**不能**。这是这次故障最常见的误解：HikariCP 的 `connectionTimeout` 控制的是"业务线程等待 getConnection 返回"的最大时长，**完全控制不到驱动内部 socket read**。要控制 socket read，必须在 JDBC URL 上显式配 `socketTimeout`。
2. "HikariCP 4.x 已修复该 bug"——不能。`async-init` 切的是应用层 init 脚本（详见 Q11），驱动握手 read 仍在 `newConnection()` 同步路径里。
3. "是 mysql 驱动版本问题，升 8.x 就好"——**不完全对**。`mysql-connector-java` 5.1.x / 8.x 在默认 URL 下都有这个问题；修复靠 URL 上显式配 `socketTimeout`，驱动升级是锦上添花而不是必选项。
4. "加 `tcpKeepAlive=true` 即可"——不够。默认探测周期 2 小时，故障窗口内救不回来；`tcpKeepAlive` 是辅助手段。

### 4.2 选型建议

| 维度 | 建议 |
|---|---|
| JDBC URL `socketTimeout` | **必填**（与驱动 / JDK / mysql 版本无关） |
| JDBC URL `connectTimeout` | 必填，典型值 3000-5000ms |
| HikariCP `connectionTimeout` | 必填，但**它控制不了驱动 socket** |
| MySQL 驱动 | ≥ 5.1.49（让 `tcpKeepAlive` 默认 true）；8.x 也行但同样要配 URL |
| 监控 | HikariPoolMXBean 全量指标 + 告警：**任何池 `total=0 && waiting>0` 持续超过 1 分钟必须告警** |

### 4.3 排查 SOP（一页纸版）

```
SQLTransientConnectionException + 多池卡死
    │
    ├── 第一步：查 JDBC URL 有没有 socketTimeout（10 秒）
    │      │
    │      ├── 没有 → ★ 一行配置即可修复，不需要升级 ★
    │      └── 有 → 进入第二步
    │
    ├── 抓 stack dump（间隔 ≥10 分钟两次）
    │      │
    │      ├── adder 线程栈底在 socketRead0 + 两次栈一致
    │      │      │
    │      │      └── 抓包确认握手 read 是否有服务端回包
    │      │             │
    │      │             ├── 有回包 → 驱动 bug（升级到 8.x）
    │      │             └── 无回包 → 服务端问题
    │      │                   │
    │      │                   ├── max_connections / Threads_connected
    │      │                   ├── skip-name-resolve
    │      │                   ├── 主从切换 / VIP 漂移
    │      │                   └── 磁盘 IO / 防火墙
    │      │
    │      └── adder 线程在等锁 / 等 FutureTask → 池满或业务并发高
    │
    └── 观察 HikariPoolMXBean：active / idle / total / waiting 时间线
```

排查手册讲到这里，方法论已经齐了。**下面两节是这次故障的原始证据**——一份故障复盘如果只讲方法、不放原始材料，读者很难判断方法的可靠性。


## 5. 附录：原始 stack dump 关键片段

### 5.1 stack.log 摘录

```
2026-08-11 20:50:13
Full thread dump Java HotSpot(TM) 64-Bit Server VM (1.8.0_xxx mixed mode):

"MAIN_HikariCP connection adder" #8230979 daemon ... waiting on condition
   TIMED_WAITING (parking) - LinkedBlockingQueue.poll

"jdbc:mysql://10.14.127.31:19030/pool-A?... connection adder" #7952751 daemon ... runnable
   at java.net.SocketInputStream.socketRead0(Native Method)
   ...
   at com.mysql.jdbc.ConnectionImpl.loadServerVariables(ConnectionImpl.java:3797)
   ...
   at com.zaxxer.hikari.pool.HikariPool$PoolEntryCreator.call(HikariPool.java:726)
```

### 5.2 stack1.log 摘录

```
2026-08-11 21:04:39
Full thread dump Java HotSpot(TM) 64-Bit Server VM (1.8.0_xxx mixed mode):

（同一组 tid #7952751/#7952621/#7952611/#7952602/#7952551/#7952367/
 #7952361/#7952206/#7952202/#7952145，栈帧完全一致）
```

### 5.3 应用依赖（故障时））

```xml
<dependency>
    <groupId>com.zaxxer</groupId>
    <artifactId>HikariCP</artifactId>
    <version>4.0.3</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.47</version>
</dependency>
```

故障现场已经摆出来了，**接下来是这次分享里我最想留下的部分——10 多个追问，含一份完整字节码证据**。这一节源自故障复盘后两次内部 tech review 的 QA，我尽量保留了当时被问到的原话和回答。

## 6. 追问与实证

### Q1：只加 `socketTimeout=2000` 能止血吗？

A：能——前提是 JDBC URL 上**显式**配这一参数。`?socketTimeout=2000` 这一行就解决本次故障，**不需要升级任何东西**。

如果 URL 上没配（多数生产环境的默认状态），`mysql-connector-java` 5.1.x / 8.x 都把 `socketTimeout` 解析成默认值 0，`MysqlIO` 构造器跳过 `setSoTimeout`，socket 默认 `SO_RCVTIMEO = 0`，握手 read 永不超时。详见 Q7 与 Q7 补充。

### Q2：HikariCP 4.x 不是已经 `async-init` 了吗？为什么仍卡住？

A：`async-init` 切的是"握手后"的逻辑。握手 read 本身仍在 `newConnection()` 内同步执行——`Driver.connect` 没返回，adder 永远不会拿到这条新建连接。详见 Q11。

### Q3：`softEvictConnections()` 能恢复吗？

A：能重启 adder，但前提是握手 read 已被某个机制打断——例如 URL 上加了 `socketTimeout`，超时抛 `SocketTimeoutException`。**没有 socketTimeout 时**，驱逐后重连仍会死等。

### Q4：能不能直接迁移至 8.x？

A：可以，但**即使迁了 8.x，URL 上没配 socketTimeout 仍会有这个问题**——5.1.x 和 8.x 的默认行为一致，都是"URL 没配就不设超时"。所以 URL 配置是不可省略的。

迁 8.x 时注意：

- 包名 `com.mysql.jdbc.*` → `com.mysql.cj.*`
- 默认时区 / SSL 行为有变化
- 部分历史参数不再支持（如 `useOldUTF8Behavior`）
- 强烈建议灰度验证

### Q5：能否不升级驱动，仅让 DBA 修复服务端？

A：可以，但客户端脆弱性依然存在——下次服务端再抖动还会卡住。客户端（URL 加 `socketTimeout`）+ 服务端协同修复是正解。

### Q6：能否用 Druid 等替代 HikariCP？

A：Druid 等主流连接池的 `createConnection` 同样同步等待驱动返回，根因同样存在。换连接池**无法消除**驱动握手 read 死等，反而会失去 HikariCP 的监控能力。

### Q7：为什么 JDBC URL 必须显式配 `socketTimeout`？

A：因为 `mysql-connector-java` 的握手 read 默认**没有超时保护**——JDBC URL 上 `socketTimeout` 参数默认是 0，`MysqlIO` 构造器在 `socketTimeout == 0` 时**直接跳过 `setSoTimeout` 调用**，socket 上保留 JDK 默认的 `SO_RCVTIMEO = 0`。这条 read 只能等服务端主动关连接才会返回，对端不响应就永久阻塞。

**这是 mysql-connector-java 5.1.x / 8.x 的统一行为**——和版本无关。

#### 一行总结

> JDBC URL 上没配 `socketTimeout` = 握手 read 永远没超时 = 服务端出问题就卡死。

修复方法只有一行：URL 后面加 `&socketTimeout=20000`。

### Q8：HikariCP `connectionTimeout` / `validationTimeout` / `keepaliveTime` 救得了吗？

A：都救不了。**这是这次故障最容易被误解的点**：

| HikariCP 参数 | 控制范围 | 能中断握手 read？ |
|---|---|---|
| `connectionTimeout` | 业务线程 `getConnection()` 的最大等待时长 | **不能** |
| `validationTimeout` | housekeeper 探测存量连接的 timeout | **不能** |
| `keepaliveTime` | 多久探测一次已有连接的活性 | **不能** |
| `maxLifetime` | 多久驱逐一个老连接 | **不能** |
| `initializationFailTimeout` | 启动期建池失败容忍窗口 | **不能** |
| 4.x `async-init` | 把"握手后 init"切到独立线程 | **不能** |

连接池的 timeout 参数管的是"池"层；JDBC driver 的 `socketTimeout` 管的是"协议"层。**两套独立机制，前者救不了后者**。

### Q10：服务端出了什么问题？

A：常见诱因（按出现概率从高到低）：

1. `max_connections` 打满：新握手被排队，客户端 read 死等
2. DNS 反向解析慢：`gethostbyaddr` 让服务端卡在 login 阶段
3. 主从切换 / VIP 漂移：包发到旧主被丢弃，客户端 read 永不 ACK
4. 磁盘 IO 抖动：MySQL 握手线程被 IO 调度阻塞
5. LB / 防火墙会话表过期：握手包过去、回包被丢弃

> 这些都是**触发条件**——配了 `socketTimeout` 后，这些诱因发生时 HikariCP 会快速抛 `SQLTransientConnectionException`（带"连接获取超时"原因），业务方第一时间能看到，而不是看到这种"卡死 10 个池 14 分钟栈帧一动不动"的诡异故障。


### Q11：为什么 HikariCP 4.x 的 `async-init` 不管用？

A：`async-init` 切出的是"握手后"的 `connection-init-sql` / `setNetworkTimeout` 等应用配置层逻辑，目的是避免 adder 被应用侧 init 脚本拖慢。但**握手 read 仍发生在 `Driver.connect` 内部**——即 `newConnection()` 的同步路径。

更深一层：**HikariCP 每个池一个 adder 线程**——adder 同步执行 `newConnection`，等不到 `Driver.connect` 返回就一直占着。当 10 个池同时进入重建握手时，10 个 adder 线程全部被卡在 native `socketRead0`，业务侧的 `getConnection()` 也就全部排队等。这就是为什么故障看起来是"10 个池同时打挂"——本质是连接池的单 adder 同步模型把"驱动层单 read 卡死"放大了成"应用层 10 个池全卡"。

所以 `async-init` 没用——握手 read 本身仍在 `newConnection()` 同步路径里，adder 仍须等待 `Driver.connect` 返回才能进入下一步。驱动卡在握手 read 时，adder 永远等不到 `Driver.connect` 返回。

**简言之**：async-init 的"async"是给应用层 init 脚本的，不是给驱动握手 read 的。


### Q12：换 Druid 等连接池能绕开吗？

A：不能。Druid / Tomcat JDBC Pool / DBCP / Vibur 等所有主流连接池的"新建连接"均同步等待驱动返回。驱动卡在握手 read 上时，任何连接池的对应线程都会同步卡住。Druid 还自带心跳与后台校验逻辑，可能让故障更复杂。

**换连接池无法消除客户端配置根因**。唯一能让池幸免的方式，是 URL 上显式配 `socketTimeout`，让 driver 在握手阶段抛 `SocketTimeoutException`。


### Q13：服务端侧可以做哪些工作？

A：服务端修复是治本之策，与客户端修复同等重要。建议 DBA 同步处理：

- `skip-name-resolve=ON`：避免握手阶段 DNS 反查拖慢连接
- `max_connections` / `back_log` 留足 buffer
- `wait_timeout` / `interactive_timeout` 与客户端 keepalive 探测周期对齐
- 主从切换 / VIP 漂移时通过 ProxySQL / MHA 自动重试
- 关键池接入 VIP 健康检查与漂移告警
- 监控 `unauthenticated user | login` 状态的连接堆积


### Q14：能批量补全 JDBC URL 的 `socketTimeout` 吗？

A：可以，而且**建议**——因为这次故障的本质就是"URL 没配 socketTimeout"。下面给一份可执行的检查清单：

- [ ] **第一步**：在数据库 / 配置中心找一份 JDBC URL，先打补丁：`&socketTimeout=20000&connectTimeout=3000&tcpKeepAlive=true`
- [ ] **第二步**：重启该池，验证握手 read 被中断（看 jstack 是否还有 `socketRead0`）
- [ ] **第三步**：全量推广
- [ ] **第四步（上线 CI 防线）**：配置 lint 规则，JDBC URL 必须包含 `socketTimeout` 和 `connectTimeout`，否则 CI 失败


## 7. 参考资料

- HikariCP#2161: Available connections in HikariPool occasionally drop to zero and not being released
- mysql-connector-java 5.1.47 字节码：`javap -c -p mysql-connector-java-5.1.47.jar`
- mysql-connector-java 5.1.49 changelog: `tcpKeepAlive` 默认值变更

## 8. 下一步该做什么

如果你的项目所有 JDBC URL 都**还没配 `socketTimeout`**，请把以下清单贴到下一个迭代里，按顺序执行：

- [ ] **最紧急**：JDBC URL 全部补 `connectTimeout=3000&socketTimeout=20000&tcpKeepAlive=true`——一行配置解决根因
- [ ] 上线 lint 规则：JDBC URL 必须包含 `socketTimeout` 和 `connectTimeout`，否则 CI 失败
- [ ] 上线 HikariPoolMXBean 全量指标（active / idle / total / waiting），告警阈值按 §3.1 的三项特征设置：**任何池 `total=0 && waiting>0` 持续超过 1 分钟必须告警**
- [ ] 故障窗口的两份 `jstack` 与 `monitor.log` 一起存档至少 90 天，便于下次做对比
- [ ] 同步建议 DBA 排查服务端诱因（见 §6 Q10），但**这次故障的根因在客户端，服务端修复不能替代 URL 配置**

如果已经在升级或排障过程中，欢迎把这次的 stack dump、JDBC URL、MySQL `SHOW PROCESSLIST` 输出贴到评论区，我帮你看看。
