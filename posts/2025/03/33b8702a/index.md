# TCP-超时重传

TCP协议是一种面向连接的有状态网络协议。对于发送的每个数据包，一旦TCP堆栈收到特定数据包的ACK，它就认为它已成功传递。

TCP使用指数退避超时重传一个未确认的数据包，最多tcp_retries2时间（默认为15），每次重传超时在TCP_RTO_MIN（200毫秒）和TCP_RTO_MAX（120秒）之间。一旦第15次重试到期（默认情况下），TCP堆栈将通知上面的层（即应用程序）断开连接。

<!--more-->

## 参数修改

tcp_retries2可以通过修改文件`/proc/sys/net/ipv4/tcp_retries2`或`sysctl net.ipv4.tcp_retries2`进行调优。

TCP_RTO_MIN和TCP_RTO_MAX的值在Linux内核中硬编码，并由以下常量定义：

```c
#define TCP_RTO_MAX ((unsigned)(120*HZ))
#define TCP_RTO_MIN ((unsigned)(HZ/5))
```

Linux2.6+使用1000毫秒的HZ，所以TCP_RTO_MIN是200毫秒，TCP_RTO_MAX是120秒。给定tcp_retries设置为15的默认值，这意味着在将断开的网络链路通知给上层（即应用程序）之前需要924.6秒，因为在最后一次（第15次）重试到期时检测到连接断开。

## RTO

真正起到限制重传次数的并不是真正的重传次数。而是以tcp_retries2为boundary，以rto_base(如TCP_RTO_MIN 200ms)为初始RTO，计算得到一个timeout值出来。如果重传间隔超过这个timeout，则认为超过了阈值。实际TCP 的 RTO 是动态计算[^1]的，也就是说:

- 如果RTT比较小，那么RTO初始值就约等于下限200ms。由于timeout总时长是924600ms，表现出来的现象刚好就是重传了15次，超过了timeout值，从而放弃TCP流
- 如果RTT较大，比如RTO初始值计算得到的是1000ms。那么根本不需要重传15次，重传总间隔就会超过924600ms。例如一个RTT=400ms的情况，当tcp_retries2=10时，仅重传了3次就放弃了TCP流

|Retransmission|RTO(ms)|Time before a timeout(sec)|Time before a timeout(min)|
|:---|:---|:---|:---|
|1|200|0.2 secs|0.0 mins|
|2|400|0.6 secs|0.0 mins|
|3|800|1.4 secs|0.0 mins|
|4|1600|3.0secs|0.1 mins|
|5|3200|6.2 secs|0.1 mins|
|6|6400|12.6 secs|0.2 mins|
|7|12800|25.4secs|0.4 mins|
|8|25600|51.0 secs|0.9 mins|
|9|51200|102.2 secs|1.7 mins|
|10|102400|204.6 secs|3.4 mins|
|11|120000|324.6 secs|5.4 mins|
|12|120000|444.6 secs|7.4 mins|
|13|120000|564.6 secs|9.4mins|
|14|120000|684.6 secs|11.4 mins|
|15|120000|804.6 secs|13.4 mins|
|16|120000|924.6 secs|15.4 mins|

## 如何定位？

### 系统指标统计

```sh
netstat -s | grep -i -E 'retrans|timeout|lost|spurious'
```

关键行(超时重传相关)
- TCPRetransSegs：所有重传段总数（含超时 + 快速重传）
- TCPTimeouts / TCPLostRetransmit：超时重传次数（RTO 触发）
- TCPFastRetrans：快速重传（3 个重复 ACK 触发，非超时）
- TCPSpuriousRtx：虚假重传（误判丢包）

如果:

- 超时重传次数持续增长 -> 存在超时抖动
- 超时占总重传比例高 → 网络延迟大、抖动大、链路不稳定

### 单连接判断

```sh
# -t: TCP; -i: 内部信息（重传、RTO、RTT）
$ ss -ti  
ESTAB      0      0      192.168.1.100:22    10.0.0.1:12345
     rto:200 rtt:1.502/0.206 ato:40 mss:1460
     cwnd:10 ssthresh:7 bytes_acked:1234
     **retrans:3/7**  # 已重传/总重传（超时+快速）
     send 567 bytes  pacing_rate 8.2Mbps
 
# ## 过滤指定 IP / 端口
ss -ti dst 10.0.0.1
ss -ti dport = :80
```

### 抓包：确认 “超时重传”（最权威）

```sh
# 抓网卡 eth0，保存到 pcap，过滤 TCP
tcpdump -i eth0 -w tcp_retrans.pcap tcp
```

wireshark 分析:
```sh
# Wireshark 打开 pcap → 筛选
tcp.analysis.retransmission
tcp.analysis.timeout_retransmission  # 仅超时重传
```

或者使用tshark 命令行：
```sh
tshark -r tcp_retrans.pcap -Y "tcp.analysis.timeout_retransmission"
```

- 同一个 seq 号 重复出现
- 两次发包间隔 ≈ RTO（常见 200ms~ 数秒）
- 无 3 个重复 ACK → 判定为 超时重传（非快速重传）

### eBPF 工具：实时追踪超时重传（低开销）

```sh
# 安装（CentOS/RHEL）
$ yum install -y bcc-tools

# 实时打印所有 TCP 重传（含超时）
$ /usr/share/bcc/tools/tcpretrans
TIME     PID IP LADDR:LPORT  T> RADDR:RPORT        STATE
00:23:02 0   4  10.1.1.1:22 R> 10.2.2.2:12345    ESTABLISHED
```
- R> 表示重传
- 配合时间差 → 间隔大的就是 超时重传

## TCP_USER_TIMEOUT

由于tcp_retries2是个系统级参数，在实际使用中，可以针对应用修改`TCP_USER_TIMEOUT`[^2]。

建议公式为(开启了 KeepAlive 的情况):

```
TCP_USER_TIMEOUT >= TCP_KEEPIDLE + TCP_KEEPINTVL * TCP_KEEPCNT
```

TCP_USER_TIMEOUT 是RFC 54288 规定的 TCP option，用来扩展 TCP RFC 7939 协议中本身的 "User Timeout" 参数（原协议不允许配置参数大小）。其用来控制已经发送，但是尚未被 ACK的数据包的存活时间，超过这个时间则会强制关闭连接。



[^1]: [RTO的计算方法(基于RFC6298和Linux 3.10)](https://perthcharles.github.io/2015/09/06/wiki-rtt-estimator/)
[^2]: [When TCP sockets refuse to die](https://blog.cloudflare.com/when-tcp-sockets-refuse-to-die/)

