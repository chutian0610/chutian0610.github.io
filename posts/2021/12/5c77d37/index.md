# docker容器中的Jdk-availableProcessors

最近在线上环境遇到一个问题，nacos客户端线程池中有96个线程在等待.一开始以为是哪里配置有误，于是检查了nacos的配置。没有发现问题。于是只能看nacos源码了.

```java
public ClientWorker(final HttpAgent agent, final ConfigFilterChainManager configFilterChainManager, final Properties properties) {
        this.agent = agent;
        this.configFilterChainManager = configFilterChainManager;

        // Initialize the timeout parameter

        init(properties);

        executor = Executors.newScheduledThreadPool(1, new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("com.alibaba.nacos.client.Worker." + agent.getName());
                t.setDaemon(true);
                return t;
            }
        });

        executorService = Executors.newScheduledThreadPool(Runtime.getRuntime().availableProcessors(), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread t = new Thread(r);
                t.setName("com.alibaba.nacos.client.Worker.longPolling." + agent.getName());
                t.setDaemon(true);
                return t;
            }
        });

        executor.scheduleWithFixedDelay(new Runnable() {
            @Override
            public void run() {
                try {
                    checkConfigInfo();
                } catch (Throwable e) {
                    LOGGER.error("[" + agent.getName() + "] [sub-check] rotate check error", e);
                }
            }
        }, 1L, 10L, TimeUnit.MILLISECONDS);
    }
```
如上面的代码，nacos长轮询线程池在初始化时使用了`Runtime.getRuntime().availableProcessors()`.而宿主机恰好是48核*2。因此判断JVM获取可用核数错误，拿到的是宿主机核数而非容器可用核数[<sup>1</sup>](#refer-anchor-1)。

<!--more-->

## availableProcessors()的源码分析
availableProcessors方法在java.lang.Runtime类中，是个native方法。需要跟到hotspot代码中调查。

```c
// Runtime.java
	// native代码
	// 返回JAVA进程可用核数
    public native int availableProcessors();
```

JDK 8u191之前的代码:

```c
// os_linux.cpp
int os::active_processor_count() {
  // Linux doesn't yet have a (official) notion of processor sets,
  // so just return the number of online processors.
  int online_cpus = ::sysconf(_SC_NPROCESSORS_ONLN);
  assert(online_cpus > 0 && online_cpus <= processor_count(), "sanity check");
  return online_cpus;
}
```

通过sysconf获取系统参数_SC_NPROCESSORS_ONLN，所以返回的是宿主机可用核数。


JDK 8u191发布了Java Improvements for Docker Containers，支持Docker容器，并添加了两个JVM参数：
- -XX:-UseContainerSupport 关闭容器支持  
- -XX:ActiveProcessorCount 手动指定可用CPU数量

JDK 8u191的代码不好找，直接看JDK 15的:

```c
// os_linux.cpp
// 如果指定了JVM参数-XX:ActiveProcessorCount, 直接返回-XX:ActiveProcessorCount的值
// 如果在容器里面，调用OSContainer::active_processor_count
// 否则，调用Linux::active_processor_count(
int os::active_processor_count() {
  // User has overridden the number of active processors
  if (ActiveProcessorCount > 0) {
    log_trace(os)("active_processor_count: "
                  "active processor count set by user : %d",
                  ActiveProcessorCount);
    return ActiveProcessorCount;
  }

  int active_cpus;
  if (OSContainer::is_containerized()) {
    active_cpus = OSContainer::active_processor_count();
    log_trace(os)("active_processor_count: determined by OSContainer: %d",
                   active_cpus);
  } else {
  	// 返回当前进程的可用核数，较之前版本增加了cpu亲缘性处理
    active_cpus = os::Linux::active_processor_count();
  }

  return active_cpus;
}

// osContainer_linux.cpp
int OSContainer::active_processor_count() {
  assert(cgroup_subsystem != NULL, "cgroup subsystem not available");
  // 调用cgroup的active_processor_count
  // cgroup是内核提供的资源隔离机制，容器化的基础
  return cgroup_subsystem->active_processor_count();
}

// cgroupSubsystem_linux.cpp
// 如果容器指定了cpu.cfs_period_us和cpu.cfs_quota_us，就用quota除以时间周期
// 如果容器指定了cpu.shares，则使用shares计算，shares是相对值
int CgroupSubsystem::active_processor_count() {
  int quota_count = 0, share_count = 0;
  int cpu_count, limit_count;
  int result;

  CachingCgroupController* contrl = cpu_controller();
  CachedMetric* cpu_limit = contrl->metrics_cache();
  if (!cpu_limit->should_check_metric()) {
    int val = (int)cpu_limit->value();
    log_trace(os, container)("CgroupSubsystem::active_processor_count (cached): %d", val);
    return val;
  }

  cpu_count = limit_count = os::Linux::active_processor_count();
  int quota  = cpu_quota();
  int period = cpu_period();
  int share  = cpu_shares();

  if (quota > -1 && period > 0) {
    quota_count = ceilf((float)quota / (float)period);
    log_trace(os, container)("CPU Quota count based on quota/period: %d", quota_count);
  }
  if (share > -1) {
    share_count = ceilf((float)share / (float)PER_CPU_SHARES);
    log_trace(os, container)("CPU Share count based on shares: %d", share_count);
  }

  if (quota_count !=0 && share_count != 0) {
  	// 如果JVM参数PreferContainerQuotaForCPUCount为true，则返回quota_count
	// 否则返回quota_count和share_count的最小值
    if (PreferContainerQuotaForCPUCount) {
      limit_count = quota_count;
    } else {
      limit_count = MIN2(quota_count, share_count);
    }
  } else if (quota_count != 0) {
    limit_count = quota_count;
  } else if (share_count != 0) {
    limit_count = share_count;
  }

  // cpu count是内核返回的可用核数
  // 返回cpu_count和limit_count的最小值
  result = MIN2(cpu_count, limit_count);
  log_trace(os, container)("OSContainer::active_processor_count: %d", result);

  // Update cached metric to avoid re-reading container settings too often
  cpu_limit->set_value(result, OSCONTAINER_CACHE_TIMEOUT);

  return result;
}
```

## Linux查看物理CPU个数、核数、逻辑CPU个数

* CPU总核数 = 物理CPU个数 * 每颗物理CPU的核数
* 总逻辑CPU数 = 物理CPU个数 * 每颗物理CPU的核数 * 超线程数

```sh
查看CPU信息（型号）
[root@AAA ~]# cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c
     96  Intel(R) Xeon(R) Platinum 8255C CPU @ 2.50GHz

## 查看物理CPU个数
[root@AAA ~]# cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
2

## 查看每个物理CPU中core的个数(即核数)
[root@AAA ~]# cat /proc/cpuinfo| grep "cpu cores"| uniq
cpu cores    : 24

## 查看逻辑CPU的个数
[root@AAA ~]# cat /proc/cpuinfo| grep "processor"| wc -l
96
```

## 参考

<div id="refer-anchor-1"></div>

- [1] [getAvailableProcessors may incorrectly report the number of cpus in Docker container](https://bugs.openjdk.java.net/browse/JDK-8140793)

