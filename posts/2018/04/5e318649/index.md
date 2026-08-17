# Java并发之线程池-ThreadPoolExecutor

Eexecutor作为灵活且强大的异步执行框架，其支持多种不同类型的任务执行策略，提供了一种标准的方法将任务的提交过程和执行过程解耦开发，基于生产者-消费者模式，其提交任务的线程相当于生产者，执行任务的线程相当于消费者，并用Runnable来表示任务，Executor的实现还提供了对生命周期的支持，以及统计信息收集，应用程序管理机制和性能监视等机制。

```mermaid
classDiagram
    class ScheduledThreadPoolExecutor
    ScheduledThreadPoolExecutor--|>ThreadPoolExecutor
    ScheduledThreadPoolExecutor..|>ScheduledExecutorService
    class ThreadPoolExecutor
    ThreadPoolExecutor--|>AbstractExecutorService
    class ForkJoinPool
    ForkJoinPool--|>AbstractExecutorService
    class AbstractExecutorService
    &lt;&lt;abstract>> AbstractExecutorService
    AbstractExecutorService..|>ExecutorService
    ScheduledExecutorService..|>ExecutorService
    class ScheduledExecutorService
    &lt;&lt;interface>> ScheduledExecutorService
     class ExecutorService
    &lt;&lt;interface>> ExecutorService
    ExecutorService..|>Executor
    class Executor
    &lt;&lt;interface>> Executor
```

<!--more-->


## Executor

Executor是顶级接口，里面只有一个方法：

```java
public interface Executor {
    void execute(Runnable command);
}
```

## ExecutorService

ExecutorService是我们经常用到的多线程执行框架的声明，源码也很少，我们逐个方法进行解释：

```java
public interface ExecutorService extends Executor {
    // 关闭线程执行框架中的线程，但效果是不再接受新线程加入，并且等待线程执行结束后关闭Executor
    void shutdown();
    // 该命令会尝试关闭正在运行中的线程，但是也仅仅是调用terminate方法然后让jdk去决定是否结束，同时该方法返回那些awaiting状态的线程组
    List<Runnable> shutdownNow();
    // 是否已经被关闭的状态
    boolean isShutdown();
    // 是否已经被中止的状态，在shutdown和shutdownnow被调用后，并且线程全部执行结束，该状态才是true，否则都是false
    boolean isTerminated();
    // 如果线程组terminate了，返回true,超时时间到了返回false。
    boolean awaitTermination(long timeout, TimeUnit unit) throws InterruptedException;
    // 提交一个Callable的回调，然后执行完成后将结果放入Future对象。
    <T> Future<T> submit(Callable<T> task);
    // 提交一个Runnable接口实现，然后result是Future返回值
    <T> Future<T> submit(Runnable task, T result);
    // 提交一个Runnable接口实现，如果执行完成，future.get()可以返回一个null
    Future<?> submit(Runnable task);
    // 执行提交的Callable集合，然后返回各自的执行结果的Future对象列表，每个元素isDone都是true
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks) throws InterruptedException;
    // 执行提交的task集合，执行完成或者timeout之后返回结果，isDone为true
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,long timeout, TimeUnit unit) throws InterruptedException;
    // 执行提交的task集合，有一个执行完成就返回结果
    <T> T invokeAny(Collection<? extends Callable<T>> tasks) throws InterruptedException, ExecutionException;
    // 有一个执行完成或者有timeout出现
    <T> T invokeAny(Collection<? extends Callable<T>> tasks, long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}
```

## AbstractExecutorService

AbstractExecutorService 是一个抽象类，它实现了ExecutorService接口。

```java
// AbstractExecutorService 提供了 newTaskFor 方法用于将 callable 和runnable 封装成 FutureTask\(RunnableFuture\）
protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
    return new FutureTask<T>(callable);
}
// submit 方法封装了对 execute 方法的调用
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}

// 所有的future 任务完成后，返回结果
public <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
    throws InterruptedException {
    if (tasks == null)
        throw new NullPointerException();
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(tasks.size());
    boolean done = false;
    try {
        for (Callable<T> t : tasks) {
            RunnableFuture<T> f = newTaskFor(t);
            futures.add(f);
            execute(f);
        }
        for (int i = 0, size = futures.size(); i < size; i++) {
            Future<T> f = futures.get(i);
            if (!f.isDone()) {
                try {
                    f.get();
                } catch (CancellationException ignore) {
                } catch (ExecutionException ignore) {
                }
            }
        }
        done = true;
        return futures;
    } finally {
        if (!done)
            for (int i = 0, size = futures.size(); i < size; i++)
                futures.get(i).cancel(true);
    }
}
```

### doInvokeAny

该方法调用了 ExecutorCompletionService;

```java
private <T> T doInvokeAny(Collection<? extends Callable<T>> tasks,boolean timed, long nanos) 
                throws InterruptedException, ExecutionException, TimeoutException {
    if (tasks == null)
        throw new NullPointerException();
    int ntasks = tasks.size();
    if (ntasks == 0)
        throw new IllegalArgumentException();
    ArrayList<Future<T>> futures = new ArrayList<Future<T>>(ntasks);
    ExecutorCompletionService<T> ecs =
        new ExecutorCompletionService<T>(this);

    // 为了更高效地实现该方法，并不会一下子执行所有任务

    try {
        ExecutionException ee = null;
        final long deadline = timed ? System.nanoTime() + nanos : 0L;
        Iterator<? extends Callable<T>> it = tasks.iterator();

        // 1. 执行一个任务
        futures.add(ecs.submit(it.next()));
        --ntasks;
        int active = 1; // 开始任务数

        for (;;) {
            // 2. 取出当前完成任务
            Future<T> f = ecs.poll();
            if (f == null) {
                // 当前没有已完成任务
                if (ntasks > 0) {
                    --ntasks;
                    // 再执行一个任务
                    futures.add(ecs.submit(it.next()));
                    ++active; // 开始任务数 +1
                }
                else if (active == 0) // 开始任务全部终止
                    break;
                else if (timed) {
                    f = ecs.poll(nanos, TimeUnit.NANOSECONDS);
                    if (f == null)
                        throw new TimeoutException();
                    nanos = deadline - System.nanoTime();
                }
                else
                    f = ecs.take(); // 没有时间限制，阻塞
            }
            if (f != null) {
                // 有任务完成或异常
                --active;
                try {
                    return f.get();
                    // 任务异常
                } catch (ExecutionException eex) {
                    ee = eex;
                } catch (RuntimeException rex) {
                    ee = new ExecutionException(rex);
                }
            }
        }

        if (ee == null)
            ee = new ExecutionException();
        throw ee;

    } finally {
        for (int i = 0, size = futures.size(); i < size; i++)
            futures.get(i).cancel(true); // 异常，终止所有任务
    }
}
```

## ExecutorCompletionService

ExecutorCompletionService 有3个属性：

```java
// 执行任务的线程池
private final Executor executor;
// 执行 Callable ，Runnable 封装为 FutureTask的工具类
private final AbstractExecutorService aes;
// 任务完成队列
private final BlockingQueue<Future<V>> completionQueue;
```

ExecutorCompletionService 会把任务封装为QueueingFuture ，QueueingFuture 覆写了FutureTask的done方法，当任务完成时，会把future放到完成队列中。

```java
private class QueueingFuture extends FutureTask<Void> {
        QueueingFuture(RunnableFuture<V> task) {
            super(task, null);
            this.task = task;
        }
        protected void done() { completionQueue.add(task); }
        private final Future<V> task;
}
```

## 线程池ThreadPoolExecutor实现原理

在ThreadPoolExecutor类中提供的构造方法：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) { 
    //...省略...
}
```

构造参数比较多，一个一个说下：

* corePoolSize：线程池中的核心线程数；
* maximumPoolSize：线程池最大线程数，它表示在线程池中最多能创建多少个线程；
* keepAliveTime：线程池中非核心线程闲置超时时长（准确来说应该是没有任务执行时的回收时间，后面会分析）；一个非核心线程，如果不干活(闲置状态)的时长超过这个参数所设定的时长，就会被销毁掉。如果设置allowCoreThreadTimeOut(boolean value)，则会作用于核心线程
* TimeUnit：时间单位。可选的单位有分钟（MINUTES），秒（SECONDS），毫秒(MILLISECONDS) 等；
* workQueue：任务的阻塞队列，缓存将要执行的Runnable任务，由各线程轮询该任务队列获取任务执行。线程池的排队策略与BlockingQueue有关。可以选择以下几个阻塞队列。
    * ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
    * LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
    * SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
    * PriorityBlockingQueue：一个具有优先级的无限阻塞队列。
* ThreadFactory：线程创建的工厂。可以进行一些属性设置，比如线程名，优先级等等，有默认实现。
* RejectedExecutionHandler：任务拒绝策略，当运行线程数已达到maximumPoolSize，队列也已经装满时会调用该参数拒绝任务，默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。以下是JDK1.5提供的四种策略。
    * AbortPolicy：直接抛出异常。
    * CallerRunsPolicy：只用调用者所在线程来运行任务。
    * DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
    * DiscardPolicy：不处理，丢弃掉。
    * 当然也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化不能处理的任务。

### 线程池的状态

```java
//用来标记线程池状态（高3位），线程个数（低29位）
//默认是RUNNING状态，线程个数为0
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
//线程个数掩码位数
private static final int COUNT_BITS = Integer.SIZE - 3;
//线程最大个数(低29位)00011111111111111111111111111111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
//（高3位）：11100000000000000000000000000000
private static final int RUNNING    = -1 << COUNT_BITS;
//（高3位）：00000000000000000000000000000000
private static final int SHUTDOWN   =  0 << COUNT_BITS;
//（高3位）：00100000000000000000000000000000
private static final int STOP       =  1 << COUNT_BITS;
//（高3位）：01000000000000000000000000000000
private static final int TIDYING    =  2 << COUNT_BITS;
//（高3位）：01100000000000000000000000000000
private static final int TERMINATED =  3 << COUNT_BITS;
// 获取高三位 运行状态
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//获取低29位 线程个数
private static int workerCountOf(int c)  { return c & CAPACITY; }
//计算ctl新值，线程状态 与 线程个数
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

其中ctl是ThreadPoolExecutor的同步状态变量。

workerCountOf()方法取得当前线程池的线程数量，算法是将ctl的值取低29位。

runStateOf()方法取得线程池的状态，算法是将ctl的值取高3位:

* RUNNING 111 表示正在运行,接受新任务并且处理阻塞队列里的任务
* SHUTDOWN 000 表示拒绝接收新的任务,但是处理阻塞队列里的任务
* STOP 001 表示拒绝接收新的任务并且不再处理任务队列中剩余的任务，并且中断正在执行的任务。
* TIDYING 010 表示所有线程已停止，准备执行terminated()方法。
* TERMINATED 011 表示已执行完terminated()方法。

当我们向线程池提交任务时，通常使用execute()方法，接下来就先从该方法开始分析。

线程池状态转换:

* RUNNING->SHUTDOWN: 显式调用shutdown()方法，或者隐式调用了finalize(),它里面调用了shutdown()方法
* RUNNING->STOP: 显式调用shutdownNow()方法
* SHUTDOWN-> STOP: 显式调用shutdownNow()方法
* SHUTDOWN -> TIDYING: 当线程池和任务队列都为空的时候
* STOP -> TIDYING: 当线程池为空的时候
* TIDYING -> TERMINATED: 当 terminated() hook 方法执行完成时候

### execute()方法

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
   /*
    * Proceed in 3 steps:
    *
    * 1. If fewer than corePoolSize threads are running, try to
    * start a new thread with the given command as its first
    * task.  The call to addWorker atomically checks runState and
    * workerCount, and so prevents false alarms that would add
    * threads when it shouldn't, by returning false.
    *
    * 2. If a task can be successfully queued, then we still need
    * to double-check whether we should have added a thread
    * (because existing ones died since last checking) or that
    * the pool shut down since entry into this method. So we
    * recheck state and if necessary roll back the enqueuing if
    * stopped, or start a new thread if there are none.
    *
    * 3. If we cannot queue task, then we try to add a new
    * thread.  If it fails, we know we are shut down or saturated
    * and so reject the task.
    */
    int c = ctl.get();
    // step 1
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    // step 2
    if (isRunning(c) && workQueue.offer(command)) {
        // 再次获取ctl的值recheck
        int recheck = ctl.get();
        // 如果当前线程池的状态不是RUNNING，并且从队列workQueue移除command成功的话，
        // 调用reject()方法拒绝任务command
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            //如果当前工作线程woker数目为0，尝试添加新的worker线程，但是不携带任务
            addWorker(null, false); // 注意此处task为null
    }
    // step 3
    else if (!addWorker(command, false))
        reject(command);
}
```

execute 方法主要流程分为3步：

1. 线程池的线程数量小于corePoolSize核心线程数量，开启核心线程执行任务。
2. 线程池的线程数量不小于corePoolSize核心线程数量，或者开启核心线程失败，尝试将任务以非阻塞的方式添加到任务队列。
3. 任务队列已满导致添加任务失败，开启新的非核心线程执行任务。

### addWorker()方法

addWorker这个方法先尝试在线程池运行状态为RUNNING并且线程数量未达上限的情况下通过CAS操作将线程池数量+1，接着在ReentrantLock同步锁的同步保证下判断线程池为运行状态，然后把Worker添加到HashSet workers中。如果添加成功则执行Worker的内部线程。

```java
 private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        //使用CAS机制尝试将当前线程数+1
        //如果是核心线程当前线程数必须小于corePoolSize 
        //如果是非核心线程则当前线程数必须小于maximumPoolSize
        //如果当前线程数小于线程池支持的最大线程数CAPACITY 也会返回失败
        for (;;) {
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    //这里已经成功执行了CAS操作将线程池数量+1，下面创建线程
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        //Worker内部有一个Thread，并且执行Worker的run方法，因为Worker实现了Runnable
        final Thread t = w.thread;
        if (t != null) {
            //这里必须同步,在状态为运行的情况下将Worker添加到workers中
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());

                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    workers.add(w);  //把新建的woker线程放入集合保存，这里使用的是HashSet
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            //如果添加成功则运行线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        //如果woker启动失败，则进行一些善后工作，比如说修改当前woker数量等等
        if (! workerStarted)
            addWorkerFailed(w); // 内部也使用了 lock
    }
    return workerStarted;
}
```

### worker

Worker是ThreadPoolExecutor的内部类，源码如下：

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
    /**
     * This class will never be serialized, but we provide a
     * serialVersionUID to suppress a javac warning.
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** Thread this worker is running in.  Null if factory fails. */
    final Thread thread;
    /** Initial task to run.  Possibly null. */
    Runnable firstTask;
    /** Per-thread task counter */
    volatile long completedTasks;

    /**
     * Creates with given first task and thread from ThreadFactory.
     * @param firstTask the first task (null if none)
     */
    Worker(Runnable firstTask) {
        setState(-1); // inhibit interrupts until runWorker
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** Delegates main run loop to outer runWorker. */
    public void run() {
        runWorker(this);
    }

    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

    void interruptIfStarted() {
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```

Worker构造方法指定了第一个要执行的任务firstTask，并通过线程池的线程工厂创建线程。可以发现这个线程的参数为this，即Worker对象，因为Worker实现了Runnable因此可以被当成任务执行，执行的即Worker实现的run方法：

### runWorker()方法

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 因为Worker的构造函数中setState(-1)禁止了中断，这里的unclock用于恢复中断-> call tryRelease
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            w.lock();
            // If pool is stopping, ensure thread is interrupted;
            // if not, ensure thread is not interrupted.  This
            // requires a recheck in second case to deal with
            // shutdownNow race while clearing interrupt
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                //该方法是个空的实现，如果有需要用户可以自己继承该类进行实现
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //真正的任务执行逻辑
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //该方法是个空的实现，如果有需要用户可以自己继承该类进行实现
                    afterExecute(task, thrown);
                }
            } finally {
                //这里设为null，也就是循环体再执行的时候会调用getTask方法
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //当指定任务执行完成，阻塞队列中也取不到可执行任务时，会进入这里，做一些善后工作
        //比如在corePoolSize跟maximumPoolSize之间的woker会进行回收
        processWorkerExit(w, completedAbruptly);
    }
}
```

这个方法是线程池复用线程的核心代码，注意Worker继承了AbstractQueuedSynchronizer，在执行每个任务前通过lock方法加锁，执行完后通过unlock方法解锁，这种机制用来防止运行中的任务被中断。在执行任务时先尝试获取firstTask，即构造方法传入的Runnable对象，然后尝试从getTask方法中获取任务队列中的任务。在任务执行前还要再次判断线程池是否已经处于STOP状态或者线程被中断。

在runWorker中，每一个Worker在getTask()成功之后都要获取Worker的锁之后运行，也就是说运行中的Worker不会中断。因为核心线程一般在空闲的时候会一直阻塞在获取Task上，也只有中断才可能导致其退出。这些阻塞着的Worker就是空闲的线程（当然，非核心线程阻塞之后也是空闲线程）。如果设置了keepAliveTime>0，那非核心线程会在空闲状态下等待keepAliveTime之后销毁，直到最终的线程数量等于corePoolSize

woker线程的执行流程就是首先执行初始化时分配给的任务，执行完成以后会尝试从阻塞队列中获取可执行的任务，如果指定时间内仍然没有任务可以执行，则进入销毁逻辑调用processWorkerExit()方法。

>注意，这里只会回收corePoolSize与maximumPoolSize直接的那部分woker

### getTask()方法

这里getTask()方法是要重点说明的，它的实现跟我们构造参数keepAliveTime存活时间有关。我们都知道keepAliveTime代表了线程池中的线程（即woker线程）的存活时间，如果到期则回收woker线程，这个逻辑的实现就在getTask中。

getTask()方法就是去阻塞队列中取任务，用户设置的存活时间，就是从这个阻塞队列中取任务等待的最大时间，如果getTask返回null，意思就是woker等待了指定时间仍然没有取到任务，此时就会跳过循环体，进入woker线程的销毁逻辑。

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // Check if queue empty only if necessary.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // Are workers subject to culling?
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //根据超时配置有两种方法取出任务
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

这个getTask()方法通过一个循环不断轮询任务队列有没有任务到来，首先判断线程池是否处于正常运行状态，根据超时配置有两种方法取出任务：

* BlockingQueue.poll 阻塞指定的时间尝试获取任务，如果超过指定的时间还未获取到任务就返回null。
* BlockingQueue.take 这种方法会在取到任务前一直阻塞。

FixedThreadPool使用的是take方法，所以会线程会一直阻塞等待任务。CachedThreadPool使用的是poll方法，也就是说CachedThreadPool中的线程如果在60秒内未获取到队列中的任务就会被终止。

### processWorkerExit

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 正常的话再runWorker的getTask方法workerCount已经被减一了
    if (completedAbruptly)
        decrementWorkerCount();
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        // 从线程池中移除超时或者出现异常的线程
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }
    // 尝试停止线程池
    tryTerminate();
    int c = ctl.get();
    // runState为RUNNING或SHUTDOWN
    if (runStateLessThan(c, STOP)) {
        // 线程不是异常结束
        if (!completedAbruptly) {
            // 线程池最小空闲数，允许core thread超时就是0，否则就是corePoolSize
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            // 如果min == 0但是队列不为空要保证有1个线程来执行队列中的任务
            if (min == 0 && !workQueue.isEmpty())
                min = 1;
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        // 1.线程异常退出
        // 2.线程池为空，但是队列中还有任务没执行，看addWoker方法对这种情况的处理
        addWorker(null, false);
    }
}
```

### tryTerminate

processWorkerExit方法中会尝试调用tryTerminate来终止线程池。这个方法在任何可能导致线程池终止的动作后执行：比如减少wokerCount或SHUTDOWN状态下从队列中移除任务。

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 以下状态直接返回：
        // 1.线程池还处于RUNNING状态
        // 2.SHUTDOWN状态但是任务队列非空
        // 3.runState >= TIDYING 线程池已经停止了或在停止了
        if (isRunning(c) || runStateAtLeast(c, TIDYING) || (runStateOf(c) == SHUTDOWN && !workQueue.isEmpty()))
            return;
        // 只能是以下情形会继续下面的逻辑：结束线程池。
        // 1.SHUTDOWN状态，这时不再接受新任务而且任务队列也空了
        // 2.STOP状态，当调用了shutdownNow方法
        // workerCount不为0则还不能停止线程池,而且这时线程都处于空闲等待的状态
        // 需要中断让线程“醒”过来，醒过来的线程才能继续处理shutdown的信号。
        if (workerCountOf(c) != 0) { // Eligible to terminate
            // runWoker方法中w.unlock就是为了可以被中断,getTask方法也处理了中断。
            // ONLY_ONE:这里只需要中断1个线程去处理shutdown信号就可以了。
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 进入TIDYING状态
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    // 子类重载：一些资源清理工作
                    terminated();
                } finally {
                    // TERMINATED状态
                    ctl.set(ctlOf(TERMINATED, 0));
                    // 继续awaitTermination
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

### shutdown

* shutdown这个方法会将runState置为SHUTDOWN，会终止所有空闲的线程，无法接受新的任务，随后等待正在执行的任务执行完成。
* shutdownNow方法将runState置为STOP。和shutdown方法的区别在于这个方法会对执行中的线程调用 Thread.interrupt() 方法，因此大部分线程将立刻被中断。之所以是大部分，而不是全部 ，是因为interrupt()方法能力有限。如果线程中没有sleep 、wait、Condition、定时锁等应用, interrupt()方法是无法中断当前的线程的。所以，ShutdownNow()并不代表线程池就一定立即就能退出，它可能必须要等待所有正在执行的任务都执行完成了才能退出。 

> shutdown调用的是interruptIdleWorkers这个方法，而shutdownNow实际调用的是Worker类的interruptIfStarted方法。

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}

private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt(); // getTask 方法能抛出一个 InterruptedException
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        interruptWorkers();
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
void interruptIfStarted() {
    Thread t;
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
        try {
            t.interrupt();
        } catch (SecurityException ignore) {
        }
    }
}
```

### 优雅关闭

查看 shutdown 和 shutdownNow 的 java doc，会发现如下的提示：

```java
shutdown() ：Initiates an orderly shutdown in which previously submitted tasks are executed, but no new tasks will be accepted.Invocation has no additional effect if already shut down.This method does not wait for previously submitted tasks to complete execution.Use {@link #awaitTermination awaitTermination} to do that.
shutdownNow()：Attempts to stop all actively executing tasks, halts the processing of waiting tasks, and returns a list of the tasks that were awaiting execution. These tasks are drained (removed) from the task queue upon return from this method.This method does not wait for actively executing tasks to terminate. Use {@link #awaitTermination awaitTermination} to do that.There are no guarantees beyond best-effort attempts to stop processing actively executing tasks. This implementation cancels tasks via {@link Thread#interrupt}, so any task that fails to respond to interrupts may never terminate.
```

两者都提示我们需要额外执行 awaitTermination 方法，仅仅执行 shutdown/shutdownNow 是不够的。

```java
public abstract class ExecutorConfigurationSupport extends CustomizableThreadFactory
      implements DisposableBean{
    @Override
    public void destroy() {
        shutdown();
    }
    /**
     * Perform a shutdown on the underlying ExecutorService.
     * @see java.util.concurrent.ExecutorService#shutdown()
     * @see java.util.concurrent.ExecutorService#shutdownNow()
     * @see #awaitTerminationIfNecessary()
     */
    public void shutdown() {
        if (this.waitForTasksToCompleteOnShutdown) {
            this.executor.shutdown();
        }
        else {
            this.executor.shutdownNow();
        }
        awaitTerminationIfNecessary();
    }
    /**
     * Wait for the executor to terminate, according to the value of the
     * {@link #setAwaitTerminationSeconds "awaitTerminationSeconds"} property.
     */
    private void awaitTerminationIfNecessary() {
        if (this.awaitTerminationSeconds > 0) {
            try {
                this.executor.awaitTermination(this.awaitTerminationSeconds, TimeUnit.SECONDS));
            }
            catch (InterruptedException ex) {
                Thread.currentThread().interrupt();
            }
        }
    }
}
```

1. 通过 waitForTasksToCompleteOnShutdown 标志来控制是想立刻终止所有任务，还是等待任务执行完成后退出。
2. executor.awaitTermination(this.awaitTerminationSeconds, TimeUnit.SECONDS)); 控制等待的时间，防止任务无限期的运行（前面已经强调过了，即使是 shutdownNow 也不能保证线程一定停止运行）。

这里给出apache-common-pool 的一个例子:

```java
executor.shutdown();
try {
    executor.awaitTermination(timeout, unit);
} catch (final InterruptedException e) {
    // Swallow
    // Significant API changes would be required to propagate this
}
executor.setCorePoolSize(0);
executor = null;
```

### 线程池饱和策略

简单介绍下有哪些饱和策略：

* AbortPolicy 为java线程池默认的阻塞策略，不执行此任务，而且直接抛出一个运行时异常，切记ThreadPoolExecutor.execute需要try catch，否则程序会直接退出。 
* DiscardPolicy 直接抛弃，任务不执行，空方法 
* DiscardOldestPolicy 从队列里面抛弃head的一个任务，并再次execute 此task。 
* CallerRunsPolicy 在调用execute的线程里面执行此command，会阻塞入口 
* 用户自定义拒绝策略（最常用）实现RejectedExecutionHandler，并自己定义策略模式 

## 线程池工厂方法

### newFixedThreadPool

定长线程池：

* 可控制线程最大并发数（同时执行的线程数）
* 超出的线程会在队列中等待

```java
// 固定线程数量的线程池
public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
    //corePoolSize跟maximumPoolSize值一样，同时传入一个无界阻塞队列
    //根据上面分析的woker回收逻辑，该线程池的线程会维持在指定线程数，不会进行回收
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>(),
                                  threadFactory);
}
```

### newCachedThreadPool

可缓存线程池：

* 线程数无限制,Integer.MAX
* 有空闲线程则复用空闲线程，若无空闲线程则新建线程
* 一定程序减少频繁创建/销毁线程，减少系统开销
* SynchronousQueue是这样 一种阻塞队列，其中每个 put 必须等待一个 take，反之亦然。同步队列没有任何内部容量.ThreadPoolExecutor 在submit 时调用的是queue的offer方法(不阻塞)。所以，只有在有线程阻塞在queue的take方法上时，才能offer 成功。

```java
public static ExecutorService newCachedThreadPool() {
    //这个线程池corePoolSize为0，maximumPoolSize为Integer.MAX_VALUE
    //意思也就是说来一个任务就创建一个woker，回收时间是60s
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

### newSingleThreadExecutor

单线程化的线程池：

* 有且仅有一个工作线程执行任务
* 所有任务按照指定顺序执行，即遵循队列的入队出队规则

```java
public static ExecutorService newSingleThreadExecutor() {
    //线程池中只有一个线程进行任务执行，其他的都放入阻塞队列
    //外面包装的FinalizableDelegatedExecutorService类实现了finalize方法，在JVM垃圾回收的时候会关闭线程池
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

### newScheduledThreadPool

支持定时以指定周期循环执行任务:

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

其中前三种线程池是ThreadPoolExecutor不同配置的实例，最后一种是ScheduledThreadPoolExecutor的实例。

最后再说说初始化线程池时线程数的选择：

如果任务是IO密集型，一般线程数需要设置2倍CPU数以上，以此来尽量利用CPU资源。
如果任务是CPU密集型，一般线程数量只需要设置CPU数加1即可，更多的线程数也只能增加上下文切换，不能增加CPU利用率。
上述只是一个基本思想，如果真的需要精确的控制，还是需要上线以后观察线程池中线程数量跟队列的情况来定。

### 自定义线程池参数优化

1. 核心和最大池大小仅在构建时设置，但也可以使用setCorePoolSize(int)和setMaximumPoolSize(int)动态更改它们。
2. 默认情况下，即使是核心线程也只是在新任务到达时才创建和启动，但是可以使用prestartCoreThread()或prestartAllCoreThreads()方法动态地重写这一点。如果使用非空队列构造线程池，可能希望预启动线程。
3. 如果当前池中有超过corePoolSize的线程，空闲时间超过keepAliveTime(请参阅getKeepAliveTime(TimeUnit))的多余的线程将被终止。这提供了一种减少资源消耗的方法。还可以使用setKeepAliveTime(long, TimeUnit)方法动态更改此参数。使用值Long.MAX_VALUE(纳秒)有效地禁止空闲线程在关闭之前终止。默认情况下，keep-alive策略仅适用于拥有多于corePoolSize线程的情况。但是方法allowCoreThreadTimeOut(boolean)也可以用于将这个超时策略应用于核心线程，只要keepAliveTime值是非零的。
4. 程序中不再引用且没有剩余线程的池将自动关闭。如果您希望确保即使用户忘记调用shutdown()也回收未引用的池，那么必须通过设置适当的保持活动时间(使用0个核心线程的下限)和/或设置allowCoreThreadTimeOut(布尔值)来安排未使用的线程最终死亡。
5. queue的选取策略：
    * 直接传递: 如果没有立即可用的线程来运行任务，则对任务进行排队的尝试将失败，因此将构造一个新线程。此策略在处理可能具有内部依赖项的请求集时能够避免锁住。直接传递通常需要无界的最大池大小，以避免拒绝新提交的任务。当命令到达的平均速度继续快于它们能够被处理的速度时，可能会出现无限制的线程增长。
    * 无界队列: 使用无界队列将导致在所有内核池大小的线程都处于繁忙状态时新任务在队列中等待。因此，不会创建超过corePoolSize的线程。当每个任务完全独立于其他任务时，这可能是合适的，因此任务不会影响其他任务的执行;尽管这种类型的排队在消除瞬时请求暴增方面很有用，但当命令平均到达速度超过处理速度时，工作队列可能会无限增长。
    * 有界队列:当使用有限的maximumpoolsize时，有界队列有助于防止资源耗尽，但调优和控制可能更困难。队列大小和最大池大小可以相互交换:使用大队列和小池可以最小化CPU使用、OS资源和上下文切换开销，但是如果任务经常阻塞可能会导致人为的低吞吐量。使用小队列通常需要更大的池大小，这会使cpu更忙，但可能会遇到不可接受的调度开销，这也会降低吞吐量。

## 总结

从任务提交的流程角度来看，对于使用线程池的外部来说，线程池的机制是这样的：

* 如果正在运行的线程数 < coreSize，马上创建线程执行该task，不排队等待；
* 如果正在运行的线程数 >= coreSize，把该task放入阻塞队列；
* 如果队列已满 && 正在运行的线程数 < maximumPoolSize，创建新的线程执行该task；
* 如果队列已满 && 正在运行的线程数 >= maximumPoolSize，线程池调用handler的reject方法拒绝本次提交。

![](threadpool.webp)

