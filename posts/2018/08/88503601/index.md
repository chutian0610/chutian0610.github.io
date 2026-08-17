# Java源码解析-Timer

(JDK version:1.8)使用Timer执行定时任务很简单：

```java
Timer timer = new Timer();
TimerTask task = new TimerTask() {
    @Override
    public void run() {
        System.out.println("hello world");
    }
};
timer.schedule(task, 10, 2000);// 延迟10ms后，每隔2s就执行一次timer task。
```

下面我们来来分析Timer的源码。

<!--more-->

## timer

```java
public class Timer {
    // 定时任务队列
    private final TaskQueue queue = new TaskQueue();
    
    // 定时器线程
    private final TimerThread thread = new TimerThread(queue);

    //  当没有对Timer的引用和queue为空时，用于帮助定时器线程平滑退出，
    private final Object threadReaper = new Object() {
        // 当GC不可达时，由于覆盖了finalize方法，会调用一次finalize方法
        protected void finalize() throws Throwable { 
            synchronized(queue) { // 注意此处加锁了
                thread.newTasksMayBeScheduled = false;
                queue.notify(); // In case queue is empty.
            }
        }
    };

    // 自增原子integer，用于构建线程名
    private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
    private static int serialNumber() {
        return nextSerialNumber.getAndIncrement();
    }

    public Timer() {
        this("Timer-" + serialNumber());
    }
    
    public Timer(boolean isDaemon) {
        this("Timer-" + serialNumber(), isDaemon);
    }

    public Timer(String name) {
        thread.setName(name);
        // Timer初始化其实就是start了TimerThread这个线程
        thread.start(); 
    }

    public Timer(String name, boolean isDaemon) {
        thread.setName(name);
        thread.setDaemon(isDaemon);
        thread.start();
    }
    // 延时执行一次task
    public void schedule(TimerTask task, long delay) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        sched(task, System.currentTimeMillis()+delay, 0);
    }
    public void schedule(TimerTask task, Date time) {
        sched(task, time.getTime(), 0);
    }
    // 延时执行周期任务
    public void schedule(TimerTask task, long delay, long period) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, System.currentTimeMillis()+delay, -period);
    }
    public void schedule(TimerTask task, Date firstTime, long period) {
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, firstTime.getTime(), -period);
    }
    public void scheduleAtFixedRate(TimerTask task, long delay, long period) {
        if (delay < 0)
            throw new IllegalArgumentException("Negative delay.");
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, System.currentTimeMillis()+delay, period);
    }

    public void scheduleAtFixedRate(TimerTask task, Date firstTime,
                                    long period) {
        if (period <= 0)
            throw new IllegalArgumentException("Non-positive period.");
        sched(task, firstTime.getTime(), period);
    }

    //真正产生延时task的地方
    private void sched(TimerTask task, long time, long period) {
        if (time < 0)
            throw new IllegalArgumentException("Illegal execution time.");

        // Constrain value of period sufficiently to prevent numeric
        // overflow while still being effectively infinitely large.
        if (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;

        synchronized(queue) {
            if (!thread.newTasksMayBeScheduled)
                throw new IllegalStateException("Timer already cancelled.");

            synchronized(task.lock) {
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException(
                        "Task already scheduled or cancelled");
                task.nextExecutionTime = time;
                task.period = period;
                task.state = TimerTask.SCHEDULED;
            }

            queue.add(task); // 添加task
            if (queue.getMin() == task)
                queue.notify(); 
                // 1. queue empty 时，thread阻塞在queue上，此时通知thread
                // 2. 更新了最近task，通知thread
        }
    }

    public void cancel() {
        synchronized(queue) {
            thread.newTasksMayBeScheduled = false;
            queue.clear();
            queue.notify();  // In case queue was already empty.
            // 防止thread wait阻塞，通知thread
        }
    }
     // 删除queue中所有cancelled task
     public int purge() {
         int result = 0;

         synchronized(queue) {
             for (int i = queue.size(); i > 0; i--) {
                 if (queue.get(i).state == TimerTask.CANCELLED) {
                     queue.quickRemove(i);
                     result++;
                 }
             }

             if (result != 0)
                 queue.heapify();
         }

         return result;
     }
    }
}
```

## timerThread

```java
class TimerThread extends Thread {

    // threadReaper 在 timer 没有任何引用时，将该标志位设为false
    boolean newTasksMayBeScheduled = true;

    // 注意，由于TimerThread 在运行时，属于GC root中的一种
    // 所以，此处保留了queue的引用，而不是Timer的引用，否则Timer永远不会被回收。
    private TaskQueue queue;

    TimerThread(TaskQueue queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            mainLoop();
        } finally {
            // Someone killed this Thread, behave as if Timer cancelled
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear();  // Eliminate obsolete references
            }
        }
    }

    private void mainLoop() {
        while (true) { // 自旋
            try {
                TimerTask task;
                boolean taskFired;
                synchronized(queue) {
                    // Wait for queue to become non-empty
                    while (queue.isEmpty() && newTasksMayBeScheduled)
                        queue.wait();
                    if (queue.isEmpty())
                        break; // Queue is empty and will forever remain; die
                        // newTasksMayBeScheduled = false 表示此时Timer可被回收

                    // Queue nonempty; look at first evt and do the right thing
                    long currentTime, executionTime;
                    task = queue.getMin();
                    synchronized(task.lock) {
                        if (task.state == TimerTask.CANCELLED) { //task 已经终止
                            queue.removeMin();
                            continue;  // No action required, poll queue again
                        }
                        currentTime = System.currentTimeMillis();
                        executionTime = task.nextExecutionTime;
                        if (taskFired = (executionTime<=currentTime)) { // 已超时，立刻执行
                            if (task.period == 0) { // Non-repeating, remove
                                queue.removeMin();
                                task.state = TimerTask.EXECUTED;
                            } else { // 周期任务，再次调度
                                queue.rescheduleMin(
                                  task.period<0 ? currentTime   - task.period   // fixed delay
                                                : executionTime + task.period); // fixed rate
                            }
                        }
                    }
                    if (!taskFired) // Task hasn't yet fired; wait
                        queue.wait(executionTime - currentTime);
                }
                if (taskFired)  // Task fired; run it, holding no locks
                    task.run(); 
                    // 注意此处执行task.run,即执行线程是当前线程TimerThread 
                    // 如果task.run执行时间太长，会导致后面的延时任务，启动时间不准，产生task堆积现象
            } catch(InterruptedException e) {
                // 该方法值只捕获线程中断异常，如果发生了其他异常，整个 Timer 就会停止。
                // 一定不要在自己的任务里抛出异常，否则一定会影响整个定时任务。
            }
        }
    }
}
```

## TaskQueue

```java
class TaskQueue {
 
    // queues是平衡二叉堆，按时间排序的最小堆
    // queue[n] 的子节点是 queue[2*n] , queue[2*n+1]
    // 堆顶是queue[1]
    private TimerTask[] queue = new TimerTask[128];

    // task 个数
    private int size = 0;
    int size() {
        return size;
    }

    void add(TimerTask task) {
        // Grow backing store if necessary
        if (size + 1 == queue.length)
            // 两倍扩容 
            queue = Arrays.copyOf(queue, 2*queue.length);
        queue[++size] = task;
        fixUp(size); 
        // 堆新增元素，向上递归
    }
    // 堆顶元素
    TimerTask getMin() {
        return queue[1];
    }

    TimerTask get(int i) {
        return queue[i];
    }

    void removeMin() {
        queue[1] = queue[size];
        queue[size--] = null;  // Drop extra reference to prevent memory leak
        fixDown(1);
        // 堆删除元素，向下递归
    }
    // 快速删除
    void quickRemove(int i) {
        assert i <= size;

        queue[i] = queue[size];
        queue[size--] = null;  // Drop extra ref to prevent memory leak
    }
    void rescheduleMin(long newTime) {
        queue[1].nextExecutionTime = newTime;
        fixDown(1);
    }

    boolean isEmpty() {
        return size==0;
    }

    void clear() {
        // Null out task references to prevent memory leak
        for (int i=1; i<=size; i++)
            queue[i] = null;

        size = 0;
    }
    // index 为 k的 树枝到根 重排
    private void fixUp(int k) {
        while (k > 1) {
            int j = k >> 1;
            if (queue[j].nextExecutionTime <= queue[k].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }
    // index 为 k的 树枝到叶子 重排
    private void fixDown(int k) {
        int j;
        while ((j = k << 1) <= size && j > 0) {
            if (j < size &&
                queue[j].nextExecutionTime > queue[j+1].nextExecutionTime)  // 找到较小的子节点
                j++; // j indexes smallest kid
            if (queue[k].nextExecutionTime <= queue[j].nextExecutionTime)
                break;
            TimerTask tmp = queue[j];  queue[j] = queue[k]; queue[k] = tmp;
            k = j;
        }
    }
    // 整个堆重排
    void heapify() {
        for (int i = size/2; i >= 1; i--)
            fixDown(i);
    }
}
```

## TimerTask

```java
public abstract class TimerTask implements Runnable {
    // 锁住task状态值
    final Object lock = new Object();

    // task的状态
    int state = VIRGIN;

    // 初始状态
    static final int VIRGIN = 0;

    // 等待执行
    static final int SCHEDULED   = 1;

    // 已执行
    static final int EXECUTED    = 2;

    // 已结束
    static final int CANCELLED   = 3;

    // 下次执行时间
    long nextExecutionTime;

    // 时间周期
    long period = 0;

    protected TimerTask() {
    }

    public abstract void run();

    public boolean cancel() {
        synchronized(lock) {
            boolean result = (state == SCHEDULED);
            state = CANCELLED;
            return result;
        }
    }
    public long scheduledExecutionTime() {
        synchronized(lock) {
            return (period < 0 ? nextExecutionTime + period
                               : nextExecutionTime - period);
        }
    }
}
```

