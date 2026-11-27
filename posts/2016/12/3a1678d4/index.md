# linux命令top

top命令是Linux下常用的性能分析工具，能够实时显示系统中各个进程的资源占用状况，top命令提供了实时的对系统处理器的状态监视.它将显示系统中CPU最“敏感”的任务列表.该命令可以按CPU使用.内存使用和执行时间对任务进行排序。
<!--more-->

## tldr

```sh
## Start top:
top

## Do not show any idle or zombie processes:
top -i

## Show only processes owned by given user:
top -u username

## Sort processes by a field:
top -o field_name

## Show the individual threads of a given process:
top -Hp process_id

## Show only the processes with the given PID(s), passed as a comma-separated list. (Normally you wouldn't know PIDs off hand. This example picks the PIDs from the process name):
top -p $(pgrep -d ',' process_name)

## Get help about interactive commands:
?
```

## usage

```sh
$ top
top - 10:31:48 up 37 days, 23:51,  7 users,  load average: 1.20, 0.94, 0.97
Tasks: 887 total,   1 running, 682 sleeping,  16 stopped,   0 zombie
%Cpu(s):  1.4 us,  0.9 sy,  0.0 ni, 97.7 id,  0.0 wa,  0.0 hi,  0.1 si,  0.0 st
KiB Mem : 26404772+total, 19938740+free, 11717728 used, 12693171+buff/cache
KiB Swap:        0 total,        0 free,        0 used. 22927366+avail Mem 
PID   USER    PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND
6514  root    20   0 4029112  46520   5276 S  14.2  0.0 369:04.71 docker-containe
24768 root    20   0 12.780g 4.461g  18600 S   7.3  1.8 616:06.92 java                                
25111 root    20   0 12.780g 4.468g  18696 S   6.0  1.8 642:51.88 java                                
5986  root    20   0 18.693g 0.996g  16376 S   3.3  0.4 294:14.76 java                      
6502  root    20   0 3156036 133556  16608 S   3.3  0.1 238:52.54 dockerd    
```

top 命令的语法如下:

```sh
top -hv | -bcHiOSs -d secs -n max -u|U user -p pid(s) -o field -w [cols]
```

|options|desc|
|:---|:---|
|-h|显示帮助信息,top的帮助信息很简略,建议查看man page|
|-v|显示版本信息|
|-c|command 列显示完整命令|
|-b|批处理|
|-i|忽略空闲任务|
|-d|top刷新间隔时间，单位秒|
|-s|保密模式|
|-S|累加模式|
|-p pid|指定进程|
|-H|线程模式|
|-n|循环显示的次数,默认是一直刷新|
|-o fieldName|以指定列名排序，还可以在列名前加上'+'或'-'来指定排序方向，例如`top -o +%CPU`|
|-O|输出top命令支持的列名|
|-w number|调节在批处理模式下的输出宽度，默认宽度依赖于命令行环境配置。|

## 详细介绍

默认前五行是当前系统情况整体的统计信息区。下面我们看每一行信息的具体意义。

* 第一行，任务队列信息，同 uptime 命令的执行结果，具体参数说明情况如下：
  * `10:31:48`: 当前系统时间
  * `up 37 days, 23:51`: 系统已经运行了37天23小时51分钟（在这期间系统没有重启过）
  * `7 users`: 当前有7个用户登录系统
  * `load average: 1.20, 0.94, 0.97`: load average后面的三个数分别是1分钟、5分钟、15分钟的负载情况。
* 第二行，Tasks — 任务（进程），具体信息说明如下:
  * 系统现在共有887个进程，其中处于运行中的有1个，682个在休眠（sleep），stoped状态的有16个，zombie状态（僵尸）的有0个。
* 第三行，cpu状态信息，具体属性说明如下：
  *  1.4 us — 用户空间占用CPU的百分比。
  *  0.9 sy — 内核空间占用CPU的百分比。
  *  0.0 ni — 改变过优先级的进程占用CPU的百分比
  * 97.7 id — 空闲CPU百分比
  *  0.0 wa — IO等待占用CPU的百分比
  *  0.0 hi — 硬中断（Hardware IRQ）占用CPU的百分比
  *  0.1 si — 软中断（Software Interrupts）占用CPU的百分比
  *  0.0 st — 虚拟机被窃取时间百分比
* 第四行,内存状态，具体信息如下：
  * 26404772 total — 物理内存总量
  * 11717728 used — 使用中的内存总量
  * 19938740 free — 空闲内存总量
  * 12693171 buffers — 缓存的内存量
* 第五行，swap交换分区信息，具体信息说明如下：
  * 0 total — 交换区总量
  * 0 used — 使用的交换区总量
  * 0 free — 空闲交换区总量
  * 22927366 avail — 可用的总内存，物理内存(不包含交换分区)。
* 第六行，空行。
* 第七行以下：各进程（任务）的状态监控，项目列信息说明如下：
  * PID — 进程id
  * USER — 进程所有者
  * PR — 进程优先级
  * NI — nice值。负值表示高优先级，正值表示低优先级
  * VIRT — 进程使用的虚拟内存总量，单位kb。VIRT=SWAP+RES
  * RES — 进程使用的、未被换出的物理内存大小，单位kb,RES=CODE+DATA
  * SHR — 共享内存大小，单位kb
  * S — 进程状态。D=不可中断的睡眠状态 R=运行 S=睡眠 T=跟踪/停止 Z=僵尸进程
  * %CPU — 上次更新到现在的CPU时间占用百分比
  * %MEM — 进程使用的物理内存百分比
  * TIME+ — 进程使用的CPU时间总计，单位1/100秒
  * COMMAND — 进程名称（命令名/命令行）

## 使用技巧

* 在top基本视图中(%Cpu(s)，显示cpu总量)，按键盘数字“1”，可监控每个逻辑CPU的状况;再按数字键1，就会返回到top基本视图界面。
* 敲击键盘“b”（打开/关闭加亮效果），高亮显示当前运行进程。
* 默认进入top时，各进程是按照CPU的占用量来排序的，敲击键盘“x”（打开/关闭排序列的加亮效果）当前排序列会被高亮。 通过"shift + >"或"shift + <"可以向右或左改变排序列。
* 在top 命令执行过程中可以使用的一些交互命令。这些命令都是单字母的，如果在命令行中使用了s 选项， 其中一些命令可能会被屏蔽。
  * h 显示帮助画面，给出一些简短的命令总结说明
  * k 终止一个进程。
  * i 忽略闲置和僵死进程。这是一个开关式命令。
  * q 退出程序
  * r 重新安排一个进程的优先级别
  * S 切换到累计模式
  * s 改变两次刷新之间的延迟时间（单位为s），如果有小数，就换算成m s。输入0值则系统将不断刷新，默认值是5 s
  * f或者F 从当前显示中添加或者删除项目
  * o或者O 改变显示项目的顺序
  * l 切换显示平均负载和启动时间信息
  * m 切换显示内存信息
  * t 切换显示进程和CPU状态信息
  * c 切换显示命令名称和完整命令行
  * M 根据驻留内存大小进行排序
  * P 根据CPU使用百分比大小进行排序
  * T 根据时间/累计时间进行排序
  * W 将当前设置写入~/.toprc文件中

