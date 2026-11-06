# HikariCP: Failed to validate connection

最近在日志中常常看到HikariCP的warnning.

```log
Failed to validate connection com.mysql.jdbc.JDBC4Connection@xxxxx (...)Possibly consider using a shorter maxLifetime value.
```
<!--more-->

## HikariCP相关配置

- minimumIdle: 池的最小连接数
- maximumPoolSize: 池的最大连接数
- idleTimeout: 允许连接在池中闲置的最长时间。**仅在minimumIdle小于maximumPoolSize时使用。池中连接小于minimumIdle时，空闲连接将不会被取消**。
- maxLifetime: 池中连接的最长生命周期。0 表示没有最大生命周期约束。
- keepaliveTime: HikariCP 连接保活的时间间隔。该值必须小于maxLifetime
- connectionTimeout: 控制客户端等待池中连接的最长毫秒数，如果在没有连接可用的情况下超过此时间，则抛出SQLException。



## mysql相关配置

- wait_timeout: mysql 为了防止空闲连接浪费，占用资源，在超过wait_timeout 时间后，会主动关闭该连接，清理资源。通过`show variables like 'wait_timeout%'`可以查询该配置值。

通过对上面配置的了解，结合日志告警，应该是maxLifeTime设置的过长了。导致池中存在已经被close的连接。所以maxLifeTime要小于空闲连接的回收时长(例如 mysql的wait_timeout )。

> 注意, **应用和数据库中间可能存在HA和DB proxy，maxLifeTime也需要考虑中间链接的空闲超时**。

