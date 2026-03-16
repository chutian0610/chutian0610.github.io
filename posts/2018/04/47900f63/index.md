# Java并发之并发优先线程池

Java原生线程池在提交任务时会优先将线程数扩展到 CoreSize，然后会将任务入队，在队列塞满后会尝试继续扩展线程数到MaxSize。这种方式适合cpu密集型任务，而且任务时间不宜过长，否则会造成队列里面任务的堆积。

对于 RPC 通信场景的 IO 密集型任务，这种调度方式就不太合适。更适合并发优先的调度策略，即优先扩展线程数到 MaxSize。然后再尝试入队。要是实现上面的调度策略需要基于 JDK 原生线程池做一下调整。

<!--more-->

## 任务队列

```java
import java.util.concurrent.LinkedBlockingQueue;
import java.util.concurrent.RejectedExecutionException;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class TaskQueue
        extends LinkedBlockingQueue<Runnable>
{
    private volatile ThreadPoolExecutor executor;

    public void setExecutor(ThreadPoolExecutor executor)
    {
        this.executor = executor;
    }

    @Override
    public boolean offer(Runnable runnable)
    {
        if (executor == null) {
            throw new RejectedExecutionException("The task queue does not have executor!");
        }

        int currentPoolThreadSize = executor.getPoolSize();
        // have free worker. put task into queue to let the worker deal with task.
        if (executor.getActiveCount() < currentPoolThreadSize) {
            return super.offer(runnable);
        }

        // return false to let executor create new worker.
        if (currentPoolThreadSize < executor.getMaximumPoolSize()) {
            return false;
        }

        // currentPoolThreadSize >= max
        return super.offer(runnable);
    }

    /**
     * Forcefully enqueue the rejected task.
     *
     * @param runnable task
     * @return offer success or not
     * @throws RejectedExecutionException if executor is terminated.
     */
    public boolean forceOffer(Runnable runnable, long timeout, TimeUnit unit)
            throws InterruptedException
    {
        if (executor == null || executor.isShutdown()) {
            throw new RejectedExecutionException("Executor is shutdown!");
        }
        return super.offer(runnable, timeout, unit);
    }
}
```

## 线程池

```java
/**
 * 并发优先的线程池
 */
public class EagerThreadPool
        extends ThreadPoolExecutor
{
    public EagerThreadPool(int corePoolSize,
            int maximumPoolSize,
            long keepAliveTime,
            TimeUnit unit,
            BlockingQueue<Runnable> workQueue,
            ThreadFactory threadFactory,
            RejectedExecutionHandler handler)
    {
        super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, threadFactory, handler);
    }

    @Override
    public void execute(Runnable command)
    {
        if (command == null) {
            throw new NullPointerException();
        }
        executeInternal(command, 0, TimeUnit.MILLISECONDS);
    }

    public void executeInternal(Runnable command, long timeout, TimeUnit unit)
    {
        try {
            super.execute(command);
        }
        catch (RejectedExecutionException rx) {
            if (getQueue() instanceof TaskQueue) {
                // If the Executor is close to maximum pool size, concurrent
                // calls to execute() may result in some tasks being rejected rather than queued.
                // If this happens, add them to the queue.
                final TaskQueue queue = (TaskQueue) getQueue();
                try {
                    if (!queue.forceOffer(command, timeout, unit)) {
                        throw new RejectedExecutionException("Queue capacity is full.", rx);
                    }
                }
                catch (InterruptedException x) {
                    throw new RejectedExecutionException(x);
                }
            }
            else {
                throw rx;
            }
        }
    }
}
```

## Tomcat 线程池 

生产中可以尝试使用 Tomcat 的线程池，并发优先且做了源码级的优化。

```xml
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>tomcat-util</artifactId>
    <version>10.1.20</version> <!-- 请根据你的Tomcat版本选择 -->
</dependency>
```

代码示例如下:

```java
import org.apache.tomcat.util.threads.ThreadPoolExecutor;
import org.apache.tomcat.util.threads.TaskQueue;
import org.apache.tomcat.util.threads.TaskThreadFactory;

import java.util.concurrent.TimeUnit;

public class TomcatThreadPoolUtil {

    private final ThreadPoolExecutor executor;

    // 在构造函数中初始化线程池
    public TomcatThreadPoolUtil(String poolName, int corePoolSize, int maximumPoolSize, int keepAliveTimeSeconds) {
        
        // 1. 创建任务队列（使用Tomcat的TaskQueue，它继承了LinkedBlockingQueue并重写了offer方法）
        TaskQueue taskQueue = new TaskQueue();
        
        // 2. 创建线程工厂，用于定义线程名、是否为守护线程等
        TaskThreadFactory threadFactory = new TaskThreadFactory(
            poolName + "-",          // 线程名前缀
            true,                    // 是否为守护线程
            Thread.NORM_PRIORITY      // 线程优先级
        );
        
        // 3. 创建Tomcat线程池
        // 注意：Tomcat的ThreadPoolExecutor构造函数与Java标准略有不同
        this.executor = new ThreadPoolExecutor(
            corePoolSize,              // 核心线程数
            maximumPoolSize,           // 最大线程数
            keepAliveTimeSeconds,      // 空闲线程存活时间
            TimeUnit.SECONDS,          // 时间单位
            taskQueue,                 // 任务队列
            threadFactory               // 线程工厂
        );
        
        // 4. 将队列与线程池关联（Tomcat TaskQueue的特殊用法）
        taskQueue.setParent(executor);
    }

    /**
     * 提交任务到线程池执行
     */
    public void submitTask(Runnable task) {
        executor.execute(task);
    }

    /**
     * 优雅关闭线程池
     */
    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    /**
     * 获取当前的线程池状态
     */
    public String getPoolStatus() {
        return String.format(
            "Pool Size: %d, Active: %d, Completed: %d, Task Queue Size: %d",
            executor.getPoolSize(),
            executor.getActiveCount(),
            executor.getCompletedTaskCount(),
            executor.getQueue().size()
        );
    }

    // ========== 使用示例 ==========
    public static void main(String[] args) {
        // 创建一个Tomcat风格线程池：核心10，最大50，空闲超时60秒
        TomcatThreadPoolUtil poolUtil = new TomcatThreadPoolUtil(
            "MyBusinessPool", 10, 50, 60
        );

        // 提交10个任务
        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            poolUtil.submitTask(() -> {
                System.out.println(Thread.currentThread().getName() + " 执行任务 " + taskId);
                try {
                    Thread.sleep(2000); // 模拟业务处理
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        // 查看线程池状态
        System.out.println(poolUtil.getPoolStatus());

        // 程序关闭时（如在ServletContextListener中）调用shutdown()
        // poolUtil.shutdown();
    }
}
```

