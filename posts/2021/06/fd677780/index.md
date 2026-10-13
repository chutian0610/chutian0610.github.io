# 获取Docker容器的内存使用情况

在之前介绍linux命令free时，有提到过docker中使用free命令获得的常常是宿主机的内存使用情况。那么怎么获取docker中内存的使用情况呢？

<!--more-->

## docker 外部

如果可以从容器外部(宿主机)监视 Docker 容器的内存使用情况。使用`docker stats`命令。

```sh
$ docker stats
CONTAINER     CPU %  MEM USAGE / LIMIT  MEM %  NET I/O     BLOCK I/O  PIDS
fc015f31d9d1  0.00%  220KiB / 3.858GiB  0.01%  1.3kB / 0B  0B / 0B    2
```

## top

docker容器实现了进程隔离，那么我们可以通过top命令获取每个进程的内存使用，从而获取整个容器的内存使用。命令如下:

```sh
top -bn 1  | awk '{ if (NR > 7) print }' |awk '{print $6}' |tr 'a-z' 'A-Z' | numfmt --from=auto |awk '{s+=$1} END {print s}' |numfmt --to=si
```

## `/sys/fs/cgroup/memory`

进入容器的 /sys/fs/cgroup/memory 这个目录, 查看 memory.stat 文件内容。

```conf
cache 2338160640
rss 7779282944
rss_huge 5068816384
mapped_file 346603520
swap 43261952
pgpgin 840197627
pgpgout 853111714
pgfault 9149670346
pgmajfault 375424
inactive_anon 399691776
active_anon 7556538368
inactive_file 1016688640
active_file 1144373248
unevictable 0
hierarchical_memory_limit 34359738368
hierarchical_memsw_limit 68719476736
total_cache 2338160640
total_rss 7779282944
total_rss_huge 5068816384
total_mapped_file 346603520
total_swap 43261952
total_pgpgin 840197627
total_pgpgout 853111714
total_pgfault 9149670346
total_pgmajfault 375424
total_inactive_anon 399691776
total_active_anon 7556538368
total_inactive_file 1016688640
total_active_file 1144373248
total_unevictable 0
```

## 参考

- [1] [Getting memory usage in Linux and Docker](https://shuheikagawa.com/blog/2017/05/27/memory-usage/)

