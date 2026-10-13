# linux命令stat系列

本文介绍 linux 常用的 stat 系列命令:
- vmstat：显示虚拟内存信息
- iostat：统计cpu和IO信息
- ifstat：阅读网络接口统计数据的便捷工具
- netstat：查看网络连接情况
- pidstat：查看进程使用信息
- dstat：系统资源统计，代替vmstat,iosat,ifstat,netstat命令。

<!--more-->

## vmstat

vmstat是Virtual Meomory Statistics（虚拟内存统计）的缩写，可对操作系统的虚拟内存、进程、CPU活动进行监控。它是对系统的整体情况进行统计，无法对某个进程进行深入分析。


### tldr

```sh
## Display virtual memory statistics:
vmstat

## Display reports every 2 seconds for 5 times:
vmstat 2 5
```

### usage

```sh
$vmstat [options] [delay [count]]
```

选项:

- `-a, --active`:显示活跃和非活跃内存
- `-f, --forks`:显示从系统启动至今的fork数量 。
- `-m, --slabs`:显示slabinfo
- `-n, --one-header`:只在开始时显示一次各字段名称。
- `-s, --stats`:显示内存相关统计信息及多种系统活动数量。
- `-d, --disk`:显示磁盘相关统计信息。
- `-D, --disk-sum`:磁盘统计汇总
- `-p, --partition <dev>`:显示指定磁盘分区统计信息
- `-S, --unit <char>`:使用指定单位显示。参数有 k 、K 、m 、M，分别代表1000、1024、1000000、1048576字节(byte)。默认单位为K(1024 bytes)
- `-w, --wide`:宽屏输出
- `-t, --timestamp`:显示时间
- `-h, --help`:显示帮助信息
- `-V, --version`:显示版本信息

### 输出

```sh
$ vmstat -w
procs -----------------------memory---------------------- ---swap-- -----io---- -system-- --------cpu--------
 r  b         swpd         free         buff        cache   si   so    bi    bo   in   cs  us  sy  id  wa  st
 4  0            0     27888808      1147684    164560880    0    0     4    29    0    0   6   4  91   0   0   
```

procs：

- r 列表示运行和等待cpu时间片的进程数。
- b 列表示在等待资源的进程数，比如正在等待I/O、或者内存交换等。

memory：

- swpd 切换到内存交换区的内存数量(k表示)。
- free 当前的空闲页面列表中内存数量(k表示)
- buff 作为buffer cache的内存数量，一般对块设备的读写才需要缓冲。
- cache: 作为page cache的内存数量，一般作为文件系统的cache，如果cache较大，说明用到cache的文件较多。

swap：

- si 由内存进入内存交换区数量
- so由内存交换区进入内存数量。

IO：

- bi 从块设备读入数据的总量（读磁盘）（每秒kb）
- bo 块设备写入数据的总量（写磁盘）（每秒kb）

system：

- in 列表示在某一时间间隔中观测到的每秒设备中断数。
- cs列表示每秒产生的上下文切换次数。

cpu：

- us 列显示了用户方式下所花费 CPU 时间的百分比。
- sy 列显示了内核进程所花费的cpu时间的百分比。
- id 列显示了cpu处在空闲状态的时间百分比
- wa 列显示了IO等待所占用的CPU时间的百分比。

## iostat

iostat是I/O statistics（输入/输出统计）的缩写，iostat工具将对系统的磁盘操作活动进行监视。它的特点是汇报磁盘活动统计情况，同时也会汇报出CPU使用情况。iostat不能对某个进程进行深入分析，仅对系统的整体情况进行分析

### tldr

```sh
## Display a report of CPU and disk statistics since system startup:
iostat

## Display a report of CPU and disk statistics with units converted to megabytes:
iostat -m

## Display CPU statistics:
iostat -c

## Display disk statistics with disk names (including LVM):
iostat -N

## Display extended disk statistics with disk names for device "sda":
iostat -xN sda

## Display incremental reports of CPU and disk statistics every 2 seconds:
iostat 2
```
### usage

```sh
$iostat [options] [delay [count]]
```

选项:

- `-c` 仅显示CPU统计信息.与-d选项互斥.
- `-d` 仅显示磁盘统计信息.与-c选项互斥.
- `-k` 以KB为单位显示每秒的磁盘请求数,默认单位块.
- `-p device | ALL` 与-x选项互斥,用于显示块设备及系统分区的统计信息.也可以在-p后指定一个设备名,如:`iostat -p hda` 或显示所有设备`iostat -p ALL`
- `-t` 在输出数据时,打印搜集数据的时间.
- `-V` 打印版本号和帮助信息.
- `-x` 输出扩展信息.
- `-m` 以 MB 为单位显示数据（默认是 KB）。
- `-t` 显示时间戳。

### 输出

```sh
$ iostat

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           6.53    0.02    3.74    0.91    0.00   88.81

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sdb              15.74       428.53        81.60 2235465539  425664008
```

cpu段:
- %user: 在用户级别运行所使用的CPU的百分比.
- %nice: nice操作所使用的CPU的百分比.
- %sys: 在系统级别(kernel)运行所使用CPU的百分比.
- %iowait: CPU等待硬件I/O时,所占用CPU百分比.
- %steal：管理程序维护另一个虚拟处理器时，虚拟CPU的无意识等待时间百分比。
- %idle: CPU空闲时间的百分比.

Device段:
- tps: 每秒钟发送到的I/O请求数.
- kB_read /s: 每秒读取KB.
- kB_wrtn/s: 每秒写入KB.
- kB_read:   读入的KB总数.
- kB_wrtn:  写入的KB总数.

```sh 
$ iostat -d -x 

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
sdb               0.30     9.36    6.16    9.58   428.51    81.60    64.82     0.01    1.19    2.84    0.12   0.08   0.13
```

- `rrqm/s`：  每秒进行 merge 的读操作数目.即 delta(rmerge)/s
- `wrqm/s`： 每秒进行 merge 的写操作数目.即 delta(wmerge)/s
- `r/s`: 每秒完成的读次数
- `w/s`: 每秒完成的写次数
- `rkB/s`: 每秒读数据量(kB为单位)
- `wkB/s`: 每秒写数据量(kB为单位)
- `avgrq-sz`:平均每次IO操作的数据量(扇区数为单位)
- `avgqu-sz`: 平均等待处理的IO请求队列长度
- `await`: 平均每次IO请求等待时间(包括等待时间和处理时间，毫秒为单位)
- `svctm`: 平均每次IO请求的处理时间(毫秒为单位)
- `%util`: 采用周期内用于IO操作的时间比率，即IO队列非空的时间比率，即一秒中有百分之多少的时间用于 I/O。如果%util接近100%，说明产生的I/O请求太多，I/O系统已经满负荷


## netstat

Netstat 是一款命令行工具，可用于列出系统上所有的网络套接字连接情况，包括 tcp, udp 以及 unix 套接字，另外它还能列出处于监听状态（即等待接入请求）的套接字。

### tldr

```sh
## To view which users/processes are listening to which ports:
sudo netstat -lnptu

## To view routing table (use -n flag to disable DNS lookups):
netstat -r

## Which process is listening to port <port>
netstat -pln | grep <port> | awk '{print $NF}'
```

### usage

```sh
netstat [option]
```

- `-a|--all` 显示所有选项，默认不显示LISTEN相关
- `-t|--tcp` 仅显示 TCP 连接
- `-u|--udp` 仅显示 UDP 连接
- `-l|--listening` 仅显示处于监听状态的端口
- `-n` 以数字形式显示 IP 地址和端口号，不进行域名解析
- `-p` 显示绑定到套接字的进程的 PID 和名称（需要管理员权限）
- `-r` 显示内核路由表
- `-i` 显示网络接口的统计信息
- `-s` 显示网络统计信息（如 TCP、UDP、ICMP 的统计

### 输出

```sh
$ sudo netstat -tulnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      1234/sshd
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      5678/cupsd
tcp6       0      0 :::80                   :::*                    LISTEN      9012/nginx
```

- Proto：协议类型（TCP 或 UDP）。
- Recv-Q/Send-Q：接收和发送队列中的数据包数量。
- Local Address：本地地址和端口。
- Foreign Address：远程地址和端口。
- State：连接状态（如 LISTEN、ESTABLISHED、TIME_WAIT 等）。
- PID/Program name：绑定到该端口的进程的 PID 和名称。

## ifstat

ifstat命令是一个用来报告网络接口的带宽使用情况的工具，它可以实时地显示每个接口的输入和输出的速率，以及总的流量量。ifstat命令可以帮助我们监控和分析网络性能，发现网络瓶颈或异常流量。

### usage

```sh
ifstat [options] [interfaces]
```

options是一些可选的参数，用来控制ifstat命令的行为和输出格式。interfaces是一个或多个网络接口的名称，用空格分隔。如果不指定接口，ifstat命令会显示所有活动的接口的信息。

- `-a`	显示所有的接口，包括没有活动的接口
- `-b`	以字节为单位显示速率，而不是以比特为单位
- `-d`	显示每个接口的描述信息
- `-h`	显示帮助信息，并退出
- `-i`	指定刷新间隔，单位是秒，默认是1秒
- `-n`	不显示接口的标题行
- `-s`	显示每个接口的总流量量，而不是速率
- `-t`	在每行的开头显示时间戳
- `-z`	不显示没有流量的接口

### 输出

```sh
$ ifstat
       eth0               lo               wlan0       
 KB/s in  KB/s out   KB/s in  KB/s out   KB/s in  KB/s out
    0.00      0.00      0.00      0.00      0.00      0.00
    0.00      0.00      0.00      0.00      0.00      0.00
    0.00      0.00      0.00      0.00      0.00      0.00
    0.00      0.00      0.00      0.00      0.00      0.00
```
ifstat命令会显示所有活动的接口的速率，单位是比特每秒，每秒刷新一次。输出的第一列是接口的名称，第二列是输入的速率，第三列是输出的速率。

## pidstat

pidstat是sysstat工具的一个命令，用于监控全部或指定进程的cpu、内存、线程、设备IO等系统资源的占用情况。pidstat首次运行时显示自系统启动开始的各项统计信息，之后运行pidstat将显示自上次运行该命令以后的统计信息。

### tldr

```sh
## Show CPU statistics at a 2 second interval for 10 times:
pidstat 2 10

## Show page faults and memory utilization:
pidstat -r

## Show input/output usage per process id:
pidstat -d

## Show information on a specific PID:
pidstat -p PID

## Show memory statistics for all processes whose command name include "fox" or "bird":
pidstat -C "fox|bird" -r -p ALL
```

### usage

```sh
pidstat [options] [delay [count]]
```

- `-u` 默认的参数，显示各个进程的cpu使用统计
- `-r` 显示各个进程的内存使用统计
- `-d` 显示各个进程的IO使用情况
- `-p` 指定进程号
- `-w` 显示每个进程的上下文切换情况
- `-t` 显示选择任务的线程的统计信息外的额外信息
- `-T { TASK | CHILD | ALL }` 这个选项指定了pidstat监控目标。
  - TASK表示报告独立的task
  - CHILD关键字表示报告进程下所有线程统计信息
  - ALL表示报告独立的task和task下面的所有线程
  - 注意：task和子线程的全局的统计信息和pidstat选项无关。这些统计信息不会对应到当前的统计间隔，这些统计信息只有在子线程kill或者完成的时候才会被收集。
- `-V` 版本号
- `-h` 在一行上显示了所有活动，这样其他程序可以容易解析。
- `-I` 在SMP环境，表示任务的CPU使用率/内核数量
- `-l` 显示命令名和所有参数

### 输出

```sh
$ pidstat

08:52:08 PM   UID       PID    %usr %system  %guest    %CPU   CPU  Command
08:52:08 PM     0         1    0.00    0.00    0.00    0.00    89  pause
08:52:08 PM     0        23    0.00    0.00    0.00    0.00    79  bash
08:52:08 PM   669        68    0.01    0.00    0.00    0.01    31  java
08:52:08 PM     0        82    0.00    0.00    0.00    0.00    41  tail
```

- PID：进程ID
- %usr：进程在用户空间占用cpu的百分比
- %system：进程在内核空间占用cpu的百分比
- %guest：进程在虚拟机占用cpu的百分比
- %CPU：进程占用cpu的百分比
- CPU：处理进程的cpu编号
- Command：当前进程对应的命令


```sh
$ pidstat -r

08:53:32 PM   UID       PID  minflt/s  majflt/s     VSZ    RSS   %MEM  Command
08:53:32 PM     0         1      0.00      0.00    1024      4   0.00  pause
08:53:32 PM     0        23      0.00      0.00   11916   2852   0.00  bash
08:53:32 PM     0        30      0.00      0.00  103904   6036   0.00  su
08:53:32 PM   669        31      0.00      0.00   10016   2976   0.00  sh
08:53:32 PM   669        68      0.69      0.00 4864744 242812   0.05  java
```

- PID：进程标识符
- Minflt/s:任务每秒发生的次要错误，不需要从磁盘中加载页
- Majflt/s:任务每秒发生的主要错误，需要从磁盘中加载页
- VSZ：虚拟地址大小，虚拟内存的使用KB
- RSS：常驻集合大小，非交换区五里内存使用KB
- Command：task命令名

```sh
$ pidstat -d

08:54:20 PM   UID       PID   kB_rd/s   kB_wr/s kB_ccwr/s  Command
08:54:20 PM   669        31      0.00      0.00      0.00  sh
08:54:20 PM   669        68      0.00      0.28      0.00  java
08:54:20 PM   669       659      0.02      1.65      0.00  java
```

- PID：进程id
- kB_rd/s：每秒从磁盘读取的KB
- kB_wr/s：每秒写入磁盘KB
- kB_ccwr/s：任务取消的写入磁盘的KB。当任务截断脏的pagecache的时候会发生。
- COMMAND:task的命令名

```sh
$ pidstat -w -p 2831
```
- PID:进程id
- Cswch/s:每秒主动任务上下文切换数量
- Nvcswch/s:每秒被动任务上下文切换数量
- Command:命令名

## dstat

dstat 命令是一个用来替换 vmstat、iostat、netstat、nfsstat 和 ifstat 这些命令的工具，通用的系统资源统计工具，是一个全能系统信息统计工具。

### tldr

```sh

## Display CPU, disk, net, paging and system statistics:
dstat

## Display statistics every 5 seconds and 4 updates only:
dstat 5 4

## Display CPU and memory statistics only:
dstat --cpu --mem

## List all available dstat plugins:
dstat --list

## Display the process using the most memory and most CPU:
dstat --top-mem --top-cpu

## Display battery percentage and remaining battery time:
dstat --battery --battery-remain
```

### usage

```sh
## dstat --list 查看 dstat 能使用的所有参数
dstat [-afv] [options] [delay [count]]
```

- `-c` 显示CPU系统占用，用户占用，空闲，等待，中断，软件中断等信息
- `-C` 可按需分别显示cpu状态
- `-d` 显示磁盘读写数据大小
- `-n` 显示网络状态
- `-N` 指定要显示的网卡
- `-l` 显示系统负载情况
- `-m` 显示内存使用情况
- `-g` 显示页面使用情况
- `-p` 显示进程状态
- `-s` 显示交换分区使用情况
- `-S` 类似D/N
- `-r` I/O请求情况
- `-y` 系统状态
- `--ipc` 显示ipc消息队列，信号等信息
- `--socket` 用来显示tcp udp端口状态
- `--output file` 把状态信息以csv的格式重定向到指定的文件中

#### 插件功能

- `-–disk-util`  显示某一时间磁盘的忙碌状况
- `-–freespace`  显示当前磁盘空间使用率
- `-–proc-count` 显示正在运行的程序数量
- `-–top-bio`    指出块I/O最大的进程
- `-–top-cpu`    图形化显示CPU占用最大的进程
- `-–top-io`     显示正常I/O最大的进程
- `-–top-mem`    显示占用最多内存的进程

### 输出

```sh
dstat
You did not select any stats, using -cdngy by default.
----total-cpu-usage---- -dsk/total- -net/total- ---paging-- ---system--
usr sys idl wai hiq siq| read  writ| recv  send|  in   out | int   csw 
  1   0  98   0   0   0|6268B 1784B|   0     0 |   0     0 |  44    39 
  0   0 100   0   0   0|   0     0 | 120B  842B|   0     0 |  50    68 
  0   0  99   1   0   0|   0     0 | 244B  362B|   0     0 |  53    61 
  1   1  98   0   0   0|   0    20k| 152B  362B|   0     0 |  54    55 
  1   0  99   0   0   0|   0     0 |  60B  362B|   0     0 |  42    54
```

- total-cpu-usage: CPU使用率
  - usr  用户空间的程序所占百分比
  - sys  系统空间程序所占百分比
  - ide  空闲百分比
  - wai  等待磁盘I/O所消耗的百分比
  - hiq  硬中断次数
  - siq  软中断次数
- dsk/total: 磁盘统计
  - read  读总数
  - writ  写总数
- net/total: 网络统计
  - recv  网络收包总数
  - send  网络发包总数
- paging: 内存分页统计
  - in  pagein（换入）
  - out page out（换出）
- system: 系统信息
  - int  中断次数
  - csw  上下文切换

## 参考

- [1][vmstat - Linux Man Page](https://www.man7.org/linux/man-pages/man8/vmstat.8.html)

