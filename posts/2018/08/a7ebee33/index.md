# Java并发之线程池-ScheduledThreadPoolExecutor

本篇是关于延时线程池的源码分析(JDK1.8)。ScheduledThreadPoolExecutor继承了ThreadPoolExecutor类，实现了ScheduledExecutorService接口。ScheduledThreadPoolExecutor 支持延后执行给定task，或是定期执行。任务在到时间后执行，但是没有任何的实时保证。对于计划执行时间相同的任务，会按照提交顺序，先进先出执行。

<!--more-->

```java
public class ScheduledThreadPoolExecutor extends ThreadPoolExecutor implements ScheduledExecutorService {}
```

## ScheduledThreadPoolExecutor

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,new DelayedWorkQueue());
}
public ScheduledThreadPoolExecutor(int corePoolSize,ThreadFactory threadFactory) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS, new DelayedWorkQueue(), threadFactory);
}
public ScheduledThreadPoolExecutor(int corePoolSize,RejectedExecutionHandler handler) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,new DelayedWorkQueue(), handler);
}
public ScheduledThreadPoolExecutor(int corePoolSize,ThreadFactory threadFactory,RejectedExecutionHandler handler) {
  super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,new DelayedWorkQueue(), threadFactory, handler);
}
```

从构造器可以看出最大线程数为Integer.MAX_VALUE，任务队列是DelayedWorkQueue,也是定时线程池实现定时功能的核心(ScheduledThreadPoolExecutor继承自ThreadPoolExecutor)。

## 添加延时任务

ScheduledThreadPoolExecutor实现的接口ScheduledExecutorService，除了继承ExecutorService的接口外,还额外实现了下面几个接口：

```java
public interface ScheduledExecutorService extends ExecutorService {
    
    ...
    // 延迟delay个时间单位后开始执行
    public ScheduledFuture<?> schedule(Runnable command,long delay, TimeUnit unit);
    // 延迟delay个时间单位后执行并返回ScheduledFuture对象
    public <V> ScheduledFuture<V> schedule(Callable<V> callable,long delay, TimeUnit unit);
    // 在initialDelay、initialDelay＋N*period分别出发,也就是执行任务时不考虑上次任务的延时。
    public ScheduledFuture<?> scheduleAtFixedRate(Runnable command,long initialDelay,long period,TimeUnit unit);
    // initialDelay开始进行，后面每次执行成功后的dealy时间单位开始,即本次执行时间会基于上次的完成时间
　　public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit);
}
```

下面是ScheduledThreadPoolExecutor 对ExecutorService 的submit方法的实现，结合ScheduledExecutorService的扩展方法。我们可以知道延时任务的创建是通过ScheduledExecutorService中的四个方法实现的。

```java
public Future<?> submit(Runnable task) {
    return schedule(task, 0, NANOSECONDS); // delay =0,立即执行
}

public <T> Future<T> submit(Runnable task, T result) {
    return schedule(Executors.callable(task, result), 0, NANOSECONDS);
}

public <T> Future<T> submit(Callable<T> task) {
    return schedule(task, 0, NANOSECONDS);
}
```

我们来依次分析下这四个方法的源码，这四个方法可以分为one-shot(一次)和period(周期)两类：

### 执行一次的延时任务

```java
// 创建一个执行一次的延时任务
public ScheduledFuture<?> schedule(Runnable command,long delay,TimeUnit unit) {
    // null check
    if (command == null || unit == null) 
        throw new NullPointerException();
    // 装饰task
    // 默认情况下, 延时线程池，使用的 ScheduledFutureTask
    // 可以通过自定义线程池覆盖decorateTask方法，返回 RunnableScheduledFuture的其它子类task
    RunnableScheduledFuture<?> t = decorateTask(command,
        // ScheduledFutureTask 的分析在下面的小节
        new ScheduledFutureTask<Void>(command, null,triggerTime(delay, unit)));
    //延迟执行
    delayedExecute(t);
    return t;
}
// callable 和runnable的 逻辑很相似
public <V> ScheduledFuture<V> schedule(Callable<V> callable,long delay,TimeUnit unit) {
    if (callable == null || unit == null)
        throw new NullPointerException();
    RunnableScheduledFuture<V> t = decorateTask(callable,
        new ScheduledFutureTask<V>(callable,triggerTime(delay, unit)));
        delayedExecute(t);
        return t;
}
```

### 周期任务

```java
// 创建周期任务，每次任务开始到下一次任务开始，时间间隔为固定值
public ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (period <= 0)
        throw new IllegalArgumentException();
    // 创建周期任务
    ScheduledFutureTask<Void> sft =
        new ScheduledFutureTask<Void>(command,null,triggerTime(initialDelay, unit),unit.toNanos(period));
        RunnableScheduledFuture<Void> t = decorateTask(command, sft);
        sft.outerTask = t;
        //延迟执行
        delayedExecute(t);
        return t;
    }
// 创建周期任务，每次任务结束到下一次任务开始，时间间隔为固定值
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,long initialDelay,long delay,TimeUnit unit) {
    if (command == null || unit == null)
        throw new NullPointerException();
    if (delay <= 0)
        throw new IllegalArgumentException();
    ScheduledFutureTask<Void> sft =
            new ScheduledFutureTask<Void>(command,null,triggerTime(initialDelay, unit),unit.toNanos(-delay)); // 此处的'-'用于区分 fixedRate和fixedDelay
    RunnableScheduledFuture<Void> t = decorateTask(command, sft);
    sft.outerTask = t;
    delayedExecute(t);
    return t;
}
```

### delayedExecute

```java
private void delayedExecute(RunnableScheduledFuture<?> task) {
    if (isShutdown())
        //如果线程池关闭了，则拒绝任务
        reject(task);
    else {
        //添加任务到队列
        super.getQueue().add(task);
        //再次检查线程池关闭 和当前任务是否能运行
        // 如果不能继续执行，将任务移出队列并取消任务。
        if (isShutdown() && !canRunInCurrentRunState(task.isPeriodic()) &&
            remove(task))
            task.cancel(false);
        else
            ensurePrestart(); //确保至少一个线程在处理任务，即使核心线程数corePoolSize为0
    }
}
// 判断是否前线程池是running 或者开启了run-after-shutdown的任务设置
boolean canRunInCurrentRunState(boolean periodic) {
        return isRunningOrShutdown(periodic ?
                    continueExistingPeriodicTasksAfterShutdown :
                    executeExistingDelayedTasksAfterShutdown);
}
// addWorker方法只在核心线程数未达上限或者没有线程的情况下执行
// 并不像ThreadPoolExecutor那样可以同时存在多个非核心线程
// ScheduledThreadPoolExecutor最多只支持一个非核心线程
void ensurePrestart() {
    int wc = workerCountOf(ctl.get());
    if (wc < corePoolSize)
        addWorker(null, true);
    else if (wc == 0)
        addWorker(null, false);
}
```

## DelayedWorkQueue

```java
static class DelayedWorkQueue extends AbstractQueue<Runnable>
        implements BlockingQueue<Runnable> {
    // 初始容量为16
    private static final int INITIAL_CAPACITY = 16;
    // 用于构建最小堆
    private RunnableScheduledFuture<?>[] queue = new RunnableScheduledFuture<?>[INITIAL_CAPACITY];
    private final ReentrantLock lock = new ReentrantLock();
    private int size = 0;
    // 用于等待堆顶task到期的线程
    // 当一个线程成为leader，只有这个线程用于等待task时间到期，其他的线程阻塞在lock上或condition的等待队列上
    private Thread leader = null;
    private final Condition available = lock.newCondition();
    ...
    public void put(Runnable e) {offer(e);}
    public boolean add(Runnable e) {return offer(e);}
    public boolean offer(Runnable e, long timeout, TimeUnit unit) {return offer(e);}
    ...
}
```

### 添加任务

通过上面的代码分析，向queue中添加task是调用了add方法,add和put方法也是调用offer,DelayedWorkQueue的offer被重写了。

```java
public boolean offer(Runnable x) {
    if (x == null)
        throw new NullPointerException();
    RunnableScheduledFuture<?> e = (RunnableScheduledFuture<?>)x;
    final ReentrantLock lock = this.lock;
    // 加锁同步
    lock.lock();
    try {
        int i = size;
        if (i >= queue.length) // 数组空间不足
            grow(); // 扩容
        size = i + 1;
        if (i == 0) {
            queue[0] = e;
            setIndex(e, 0); // 设置第一个task的heap index为0
        } else {
            siftUp(i, e);
        }
        if (queue[0] == e) { // 第一个task，或是当前任务是最早的task
            leader = null; // 重置leader为null
            available.signal(); // 重新竞争
        }
        } finally {
            lock.unlock();
        }
    return true;
}
private void grow() {
    int oldCapacity = queue.length;
    // 扩充为 原来的1.5倍
    int newCapacity = oldCapacity + (oldCapacity >> 1); // grow 50%
    if (newCapacity < 0) // overflow
            newCapacity = Integer.MAX_VALUE; // 处理溢出，最大size为 max Int 
    queue = Arrays.copyOf(queue, newCapacity); // 数组copy，heap index 不变
}
private void setIndex(RunnableScheduledFuture<?> f, int idx) {
    if (f instanceof ScheduledFutureTask) // 只处理 ScheduledFutureTask
        ((ScheduledFutureTask)f).heapIndex = idx; // 此处的heapIndex 是为了快速定位task
}
// 最小堆重排(k 向上到根)
private void siftUp(int k, RunnableScheduledFuture<?> key) {
    while (k > 0) {
        int parent = (k - 1) >>> 1;
        RunnableScheduledFuture<?> e = queue[parent];
        if (key.compareTo(e) >= 0)
            break;
        queue[k] = e; // 交换父节点和当前节点位置
        setIndex(e, k); // 更新父节点的heap index
        k = parent; // 递归至堆顶
    }
    queue[k] = key;
    setIndex(key, k); //在堆中的位置
}
```

下面我们来看下，如何从DelayedWorkQueue中获取暂存的task.

### poll

```java
// 无阻塞poll
public RunnableScheduledFuture<?> poll() {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        RunnableScheduledFuture<?> first = queue[0];
        if (first == null || first.getDelay(NANOSECONDS) > 0)
            return null;
        else
            return finishPoll(first);
        } finally {
            lock.unlock();
     }
}
public RunnableScheduledFuture<?> poll(long timeout, TimeUnit unit)throws InterruptedException {
    long nanos = unit.toNanos(timeout);
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly(); // 响应中断的可重入锁
    try {
        for (;;) {
            RunnableScheduledFuture<?> first = queue[0]; // 最小堆就是启动延时最小的
            if (first == null) { //空队列
                if (nanos <= 0) // 超时返回 null
                    return null;
                else
                    nanos = available.awaitNanos(nanos);// 限时阻塞
                    // 注意，此时让出了锁
            } else {
                long delay = first.getDelay(NANOSECONDS);
                if (delay <= 0) // 已到启动时间  
                    return finishPoll(first);
                if (nanos <= 0) // 未到启动时间，但是线程等待超时
                    return null;
                first = null; // don't retain ref while waiting
                // leader线程存在或者nanos小于于delay的情况下
                // 1. 即便等待nanos(nanos>delay)，因为leader线程到时间之后会发出signal，此时会重新竞争
                // 2. nanos小于delay，不会超时
                if (nanos < delay || leader != null) 
                    nanos = available.awaitNanos(nanos); //超时时间内，没有任务到期
                    //  注意，此时让出了锁
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
                        long timeLeft = available.awaitNanos(delay); // 超时时间内，有任务到期
                        //  注意，此时让出了锁
                        nanos -= delay - timeLeft; // 更新超时时间
                    } finally {
                        if (leader == thisThread) // leader 未丢失
                            leader = null;// 重置leader
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null) // leader 丢失
            available.signal();
        lock.unlock();
    }
}
private RunnableScheduledFuture<?> finishPoll(RunnableScheduledFuture<?> f) {
    int s = --size;
    RunnableScheduledFuture<?> x = queue[s];
    queue[s] = null;
    if (s != 0)
        siftDown(0, x); // 移除堆头，将堆尾元素放到堆头，重排堆
    setIndex(f, -1); // 重置获取task的heap index
    return f;
}
// 删除堆头，最小堆重排(k 向下到叶子)
private void siftDown(int k, RunnableScheduledFuture<?> key) {
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1;
        RunnableScheduledFuture<?> c = queue[child];
        int right = child + 1;
        if (right < size && c.compareTo(queue[right]) > 0)
            c = queue[child = right];
        if (key.compareTo(c) <= 0)
            break;
        queue[k] = c;
        setIndex(c, k);
        k = child;
    }
    queue[k] = key;
    setIndex(key, k);
}
```

### take

```java
public RunnableScheduledFuture take() throws InterruptedException {
    final ReentrantLock lock = this.lock;
    lock.lockInterruptibly();
    try {
        for (;;) {
			// 队头元素 queue[0], 就是要执行的任务
            RunnableScheduledFuture first = queue[0];
			// 任务为null, 说明此时还没有任务被提交，所以 wait.
            if (first == null)
                available.await();
            else {
				// 任务不为空，则获取当前时间 到 任务开始执行的时间（getDelay）
				// 的时间间隔 delay
                long delay = first.getDelay(TimeUnit.NANOSECONDS);
				// 如果 delay <= 0 表示，任务已经可以运行了，所以调用
				// finishPoll(first) 方法将 first 出队列，获取到 first
				// 的 worker 线程将开始执行任务。
                if (delay <= 0)
                    return finishPoll(first);
                else if (leader != null)
                    available.await();
                else {
                    Thread thisThread = Thread.currentThread();
                    leader = thisThread;
                    try {
						// 开始进入等待状态，等待的时间就是 delay
						// 从逻辑上看，这个等待返回之后，delay 将
						// <= 0，所以 first 任务可以被调度了
						// 所以这个循环将在上面的 return finishPoll(first);
						// 中退出。
                        available.awaitNanos(delay);
                    } finally {
                        if (leader == thisThread)
                            leader = null;
                    }
                }
            }
        }
    } finally {
        if (leader == null && queue[0] != null)
            available.signal();
        lock.unlock();
    }
}
```

#### take 过程中leader thread的说明

假设，有 T1, T2, T3 三个 worker 线程，在竞争获取 queue[0] 任务。

* 当 T1 成功执行到 take 的循环中的最后一个 else 中的 available.awaitNanos(delay); 线程时， leader 就会被设置成 T1。
* 此时处于 wait 过程的线程 T1, 将放弃锁 lock, 所以 T2, T3 又开始竞争锁
* 假设 T2 获得锁，进行循环执行到 else if (leader != null) ，显然 leader 已经是 T1 了，所以这个判断成立，所以 T2 会进入 available.await();
* 此时 T1 因为 available.awaitNanos(delay) delay 时间已过，所以被唤醒，重新获取锁之后，将 leader 置为 null, 在 T1 返回之前，执行最后的 finally 语句，将通知 像 T2 这样的线程（处于 available.await()），使得 T2 可以立即参与到下一次获取 queue[0] 的过程中。
* 此时，T1 获取了任务，正在执行，T2 被 T1 唤醒，T2 和 T3 又开始了第一个步骤中的过程一样进行任务获取了。

通过这三个 worker 线程获取任务的过程可知：leader 的使用，使得 worker 线程的 wait 最小化，尽量使得 worker 能够参与到任务的获取和执行中来。如果不使用 leader， 则 T1, T2, T3 线程最终都进入 available.awaitNanos(delay); 的过程，而对于一个任务来说，多个线程在其上等待是没有意义的，因为最终只需要一个线程来执行任务，而不是所有在其上 wait 的线程，所以 leader 其实也可以认为是当前queue[0]最终可以被获成功的那个线程。显然应该只有一个

## ScheduledFutureTask

```mermaid
classDiagram
    ScheduledFutureTask --|>FutureTask
    ScheduledFutureTask ..|> RunnableScheduledFuture
    FutureTask ..|> RunnableFuture
    RunnableScheduledFuture..|> RunnableFuture
    RunnableScheduledFuture..|> ScheduledFuture
    ScheduledFuture..|> Future
    ScheduledFuture..|> Delayed
    RunnableFuture..|> Future
    RunnableFuture..|> Runnable
    &lt;&lt;interface>> RunnableScheduledFuture
    &lt;&lt;interface>> RunnableFuture
    &lt;&lt;interface>> ScheduledFuture
    &lt;&lt;interface>> Future
    &lt;&lt;interface>> Delayed
    &lt;&lt;interface>> Runnable
```

### ScheduledFutureTask 实现的接口

```java

public interface Delayed extends Comparable<Delayed> {
    // 返回对象剩余的延迟时间
    long getDelay(TimeUnit unit);
}

public interface ScheduledFuture<V> extends Delayed, Future<V> {}

public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}

public interface RunnableScheduledFuture<V> extends RunnableFuture<V>, ScheduledFuture<V> {
    // 是否是定时任务
    boolean isPeriodic();
}
```

### ScheduledFutureTask 源码分析

```java
// ScheduledFutureTask 是ScheduledThreadPoolExecutor的内部类
private class ScheduledFutureTask<V> extends FutureTask<V> implements RunnableScheduledFuture<V> {

  /** 自增的任务序号 */
  private final long sequenceNumber;

  /** 任务执行时间，纳秒为单位 */
  private long time;

  /**
    * 纳秒为单位的任务周期执行间隔；
    * 正数表示 fixed-rate execution.  
    * 负数表示 fixed-delay execution.
    * 0 表示不重复任务.
    */
  private final long period;

  /** The actual task to be re-enqueued by reExecutePeriodic */
  RunnableScheduledFuture<V> outerTask = this;

  /** 任务队列的索引，用于支持快速cancel(实际上是用于cancel中的remove)*/
  int heapIndex;
  // 单次执行的任务
  ScheduledFutureTask(Runnable r, V result, long ns) {
      super(r, result);
      this.time = ns;
      this.period = 0;
      /* static final AtomicLong sequencer = new AtomicLong();
         ScheduledThreadPoolExecutor 类属性，全局唯一*/
      this.sequenceNumber = sequencer.getAndIncrement();
  }
  // 周期任务
  ScheduledFutureTask(Runnable r, V result, long ns, long period) {
      super(r, result);
      this.time = ns;
      this.period = period;
      this.sequenceNumber = sequencer.getAndIncrement();
  }
  // 单次执行任务，注意callable 任务不存在周期任务
  ScheduledFutureTask(Callable<V> callable, long ns) {
      super(callable);
      this.time = ns;
      this.period = 0;
      this.sequenceNumber = sequencer.getAndIncrement();
  }
  // Delayed 实现方法，获取距离执行时间还有多久
  public long getDelay(TimeUnit unit) {
      return unit.convert(time - now(), NANOSECONDS);
  }

 /**
   * ScheduledFutureTask在等待队列里调度不再按照FIFO，
   * 而是按照执行时间，谁即将执行，谁就排在前面。
   * 在这里也可以看到sequenceNumber的作用，当执行时间相同时，按照序号排序。
   */

  // 实现Comparable<T>接口
  public int compareTo(Delayed other) {
      if (other == this) // compare zero if same object
          return 0;
      if (other instanceof ScheduledFutureTask) {
          ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
          long diff = time - x.time;
          if (diff < 0)
              return -1;
          else if (diff > 0)
              return 1;
          else if (sequenceNumber < x.sequenceNumber) // 开始时间相同时，比较任务序号
              return -1;
          else
              return 1;
      }
      long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS); // 比较执行时间
      return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
  }

  // 判断是否周期操作
  public boolean isPeriodic() {
      return period != 0;
  }
  // 设置下次执行时间
  private void setNextRunTime() {
      long p = period;
      if (p > 0)
          time += p; // fix rate
      else
          time = triggerTime(-p); // fix delay
          // 从当前开始计算延时
  }

// ====================================================================================
    // triggerTime 是外部类 ScheduledThreadPoolExecutor的方法
    // long的取值范围为（-9223372036854774808~9223372036854774807）
    long triggerTime(long delay) {
        return now() +
            ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay)); 
            // 防止delay溢出，限制在 Long.MAX_VALUE内
    }
// ====================================================================================

  public boolean cancel(boolean mayInterruptIfRunning) {
      boolean cancelled = super.cancel(mayInterruptIfRunning);
      // removeOnCancel 是ScheduledThreadPoolExecutor的配置
      // removeOnCancel 表示task被cancel时，是否需要从queue中删除，默认为false
      if (cancelled && removeOnCancel && heapIndex >= 0)
          // 从工作队列中删除
          remove(this);
      return cancelled;
  }
  /**
    * Overrides FutureTask version so as to reset/requeue if periodic.
    */
  public void run() {
      boolean periodic = isPeriodic();
      // 当前线程池状态是否可以执行任务，shutdown后，是否继续执行
      if (!canRunInCurrentRunState(periodic))
          cancel(false);
          // 如果执行的是非周期型任务，调用ScheduledFutureTask.super.run()方法
      else if (!periodic)
          ScheduledFutureTask.super.run();
          // 如果执行的是周期型任务，则执行ScheduledFutureTask.super.runAndReset():
      else if (ScheduledFutureTask.super.runAndReset()) {
          setNextRunTime();
          // 将任务再次添加到任务队列:
          reExecutePeriodic(outerTask);
      }
  }
   // ====================== ScheduledThreadPoolExecutor 的方法 ========================
   // ScheduledFutureTask 是 ScheduledThreadPoolExecutor 的内部类
   void reExecutePeriodic(RunnableScheduledFuture<?> task) {
        if (canRunInCurrentRunState(true)) {
            super.getQueue().add(task);
            if (!canRunInCurrentRunState(true) && remove(task))
                task.cancel(false);
            else
                ensurePrestart();
        }
    }
    //===============================================================================
}
```

