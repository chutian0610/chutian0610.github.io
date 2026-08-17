# Java引用

在JDK1.2之前，Java中的引用的定义很传统: 如果reference类型(栈中的引用类型)的数据中存储的数值代表另一块内存的起始地址，就称这块内存代表一个引用。在这种情况下，对象只有被引用和没有被引用两种状态。

我们希望有这样一类对象，当内存空间还足够时，保存在内存中；而如果内存空间在垃圾回收后还是紧张，则可以抛弃这些对象。

JDK1.2后，java对引用的概念进行了扩充，将引用分为强引用\(Strong Reference\)、软引用\(Soft Reference\)、弱引用\(Weak Reference\)、虚引用\(Phanton Reference\)这四种，引用强度依次减弱。主要有两个目的：第一是可以让程序员通过代码的方式决定某些对象的生命周期；第二是有利于JVM进行垃圾回收。

<!--more-->

## 扩展的引用类型

jvm有四种引用: strong soft weak phantom(其实还有一种FinalReference，这个由jvm自己使用，外部无法调用到），主要的区别体现在gc上的处理，如下：

* Strong类型，也就是正常使用的类型，不需要显示定义，只要没有任何引用就可以回收
* SoftReference类型，如果一个对象只剩下一个soft引用，在jvm内存不足的时候会将这个对象进行回收
* WeakReference类型，如果对象只剩下一个weak引用，那gc的时候就会回收。和SoftReference都可以用来实现cache
* PhantomReference类型，如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收，可以用来实现类似Object.finalize功能。

### 强引用

强引用就是指在程序代码之中普遍存在的，比如下面这段代码中的object和str都是强引用：

```java
Object object = new Object();
String str = "hello";
```

只要某个对象有强引用与之关联，JVM必定不会回收这个对象，即使在内存不足的情况下，JVM宁愿抛出OutOfMemory错误也不会回收这种对象。比如下面这段代码：

```java
public class Main {
    public static void main(String[] args) {
        new Main().fun1();
    }
    public void fun1() {
        Object object = new Object();
        Object[] objArr = new Object[1000];
    }
}
```

当运行至`Object[] objArr = new Object[1000];`这句时，如果内存不足，JVM会抛出OOM错误也不会回收object指向的对象。不过要注意的是，当fun1运行完之后，object和objArr都已经不存在了，所以它们指向的对象都会被JVM回收。

如果想中断强引用和某个对象之间的关联，可以显示地将引用赋值为null，这样一来的话，JVM在合适的时间就会回收该对象。比如Vector类的clear方法中就是通过将引用赋值为null来实现清理工作的：

```java
    public synchronized E remove(int index) {
    modCount++;
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);
    Object oldValue = elementData[index];

    int numMoved = elementCount - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index,
                 numMoved);
    elementData[--elementCount] = null; // Let gc do its work

    return (E)oldValue;
    }
```

### 软引用

软引用是用来描述一些有用但并不是必需的对象，在Java中用`java.lang.ref.SoftReference`类来表示。对于软引用关联着的对象，只有在内存不足异常抛出前，JVM才会把该对象列入回收范围中进行二次回收。若内存仍不足，会抛出内存溢出异常。因此，这一点可以很好地用来解决OOM的问题，并且这个特性很适合用来实现缓存：比如网页缓存、图片缓存等。

```java
import java.lang.ref.SoftReference;

public class Main {
    public static void main(String[] args) {
        SoftReference<String> sr = new SoftReference<String>(new String("hello"));
        System.out.println(sr.get());
    }
}
```

### 弱引用

弱引用也是用来描述非必需对象的，当JVM进行垃圾回收时，无论内存是否充足，都会回收被弱引用关联的对象。在java中，用`java.lang.ref.WeakReference`类来表示。下面是使用示例：

```java
import java.lang.ref.WeakReference;

public class Main {
    public static void main(String[] args) {
        WeakReference<String> sr = new WeakReference<String>(new String("hello"));
        System.out.println(sr.get());
        System.gc();                //通知JVM的gc进行垃圾回收
        System.out.println(sr.get());
    }
}
/** ------ out put ----
 * hello
 * null
 */
```

第二个输出结果是null，这说明只要JVM进行垃圾回收，被弱引用关联的对象必定会被回收掉。弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被JVM回收，这个软引用就会被加入到与之关联的引用队列中。

>要注意的是，这里所说的被弱引用关联的对象是指只有弱引用与之关联，如果存在强引用同时与之关联，则进行垃圾回收时也不会回收该对象（软引用也是如此）。

### 虚引用

虚引用和前面的软引用、弱引用不同，它并不影响对象的生命周期。在java中用java.lang.ref.PhantomReference类表示。如果一个对象与虚引用关联，则跟没有引用与之关联一样，在任何时候都可能被垃圾回收器回收。要注意的是，虚引用必须和引用队列关联使用。

当referent被gc回收时，JVM自动把PhantomReference对象(reference)本身加入到ReferenceQueue中，像发出信号通知一样，表明该reference指向的referent被回收。然后可以通过去queue中取到reference，此时说明其指向的referent已经被回收，可以通过这个通知机制来做额外的清场工作。 因此有些情况可以用PhantomReference 代替finalize()，做资源释放更明智。

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;
public class Main {
    public static void main(String[] args) {
        ReferenceQueue<String> queue = new ReferenceQueue<String>();
        PhantomReference<String> pr = new PhantomReference<String>(new String("hello"), queue);
        System.out.println(pr.get());
    }
}
```

使用虚引用有潜在的内存泄露风险，因为JVM不会自动帮助我们释放，我们必须要保证它指向的堆对象是不可达的。

* 软引用和弱引用差别不大，JVM都是先将其referent字段设置成null，之后将软引用或弱引用，加入到关联的引用队列中。我们可以认为JVM先回收堆对象占用的内存，然后才将软引用或弱引用加入到引用队列。  
* 而虚引用则不同，JVM不会自动将虚引用的referent字段设置成null，而是先保留堆对象的内存空间，直接将PhantomReference加入到关联的引用队列，也就是说如果我们不手动调用PhantomReference.clear()，虚引用指向的堆对象内存是不会被释放的。 

## Reference与ReferenceQueue

Reference作为SoftReference，WeakReference，PhantomReference，FinalReference这几个引用类型的父类。主要有两个字段referent、queue，一个是指所引用的对象，一个是与之对应的ReferenceQueue。Reference类有个构造函数 `Reference(T referent, ReferenceQueue<? super T> queue)`，可以通过该构造函数传入与Reference相伴的ReferenceQueue。

ReferenceQueue本身提供队列的功能，有入队(enqueue)和出队(poll,remove,其中remove阻塞等待提取队列元素)。ReferenceQueue对象本身保存了一个Reference类型的head节点，Reference封装了next字段，这样就是可以组成一个单向链表。同时ReferenceQueue提供了两个静态字段NULL，ENQUEUED。

```java
static ReferenceQueue<Object> NULL = new Null<>();
static ReferenceQueue<Object> ENQUEUED = new Null<>();
```

这两个字段的主要功能：NULL是当我们构造Reference实例时queue传入null时，会默认使用NULL，这样在enqueue时判断queue是否为NULL,如果为NULL直接返回，入队失败。ENQUEUED的作用是防止重复入队，reference后会把其queue字段赋值为ENQUEUED,当再次入队时会直接返回失败。

```java
boolean enqueue(Reference<? extends T> r) { /* Called only by Reference class */
    synchronized (lock) {
        // Check that since getting the lock this reference hasn’t already been
        // enqueued (and even then removed)
        ReferenceQueue<?> queue = r.queue;
        // 此处的 NULL 是上方的 ReferenceQueue<Object> NULL 
        if ((queue == NULL) || (queue == ENQUEUED)) {
            return false;
        }
        assert queue == this;
        r.queue = ENQUEUED; // reference 入队后，状态改为入队
        r.next = (head == null) ? r : head;
        head = r;// 插入队列头部
        queueLength++;
        if (r instanceof FinalReference) {
            sun.misc.VM.addFinalRefCount(1);
        }
        lock.notifyAll();
        return true;
    }
}
```

### Reference类

Reference类存在四种不同的状态：

* Active：接受垃圾收集器的特殊处理。 在收集器检测到引用对象的可达性已更改为适当状态后的一段时间，它会将实例的状态更改为 Pending 或 Inactive，具体取决于实例在创建时是否已注册到队列中。 在前一种情况下，它还将实例添加到待处理的引用列表中。 新创建的实例处于活动状态。
* Pending：待处理引用列表的一个元素，等待被引用处理线程加入到ReferenceQueue。 未注册的实例永远不会处于这种状态。
* Enqueued：实例创建时注册了队列元素。 当一个实例从其 ReferenceQueue 中移除时，它会变为非活动状态。 未注册的实例永远不会处于这种状态。
* Inactive：无事可做。 一旦一个实例变为非活动状态，它的状态将永远不会再改变，等待gc回收。


```java
// public abstract class Reference<T> 
   // 引用的对象
    private T referent;         /* Treated specially by GC */

    volatile ReferenceQueue<? super T> queue;
   /* When 
    *    Active:   NULL
    *    pending:   this
    *    Enqueued:   next reference in queue (or this if last)
    *    Inactive:   this
    */
    Reference next; 

   /* When 
     *    active:   next element in a discovered reference list maintained by GC (or this if last)
     *    pending:   next element in the pending list (or null if last)
     *    otherwise:   NULL
     */
    transient private Reference<T> discovered;  /* used by VM */

    /* Object used to synchronize with the garbage collector.  The collector
     * must acquire this lock at the beginning of each collection cycle.  It is
     * therefore critical that any code holding this lock complete as quickly
     * as possible, allocate no new objects, and avoid calling user code.
     */
    static private class Lock { };
    private static Lock lock = new Lock();

    /* List of References waiting to be enqueued.  The collector adds
     * References to this list, while the Reference-handler thread removes
     * them.  This list is protected by the above lock object. The
     * list uses the discovered field to link its elements.
     */
    private static Reference<Object> pending = null;

    //... ...
    private static class ReferenceHandler extends Thread {

    ReferenceHandler(ThreadGroup g, String name) {
        super(g, name);
    }

    public void run() {
        while (true) {
            tryHandlePending(true);
        }
    }
}

    static boolean tryHandlePending(boolean waitForNotify) {
        Reference<Object> r;
        Cleaner c;
        try {
            synchronized (lock) {
                if (pending != null) {
                    r = pending;
                    // 'instanceof' might throw OutOfMemoryError sometimes
                    // so do this before un-linking 'r' from the 'pending' chain...
                    c = r instanceof Cleaner ? (Cleaner) r : null;
                    // unlink 'r' from 'pending' chain
                    pending = r.discovered;
                    r.discovered = null;
                } else {
                    // The waiting on the lock may cause an OutOfMemoryError
                    // because it may try to allocate exception objects.
                    if (waitForNotify) {
                        lock.wait();
                    }
                    // retry if waited
                    return waitForNotify;
                }
            }
        } catch (OutOfMemoryError x) {
            // Give other threads CPU time so they hopefully drop some live references
            // and GC reclaims some space.
            // Also prevent CPU intensive spinning in case 'r instanceof Cleaner' above
            // persistently throws OOME for some time...
            Thread.yield();
            // retry
            return true;
        } catch (InterruptedException x) {
            // retry
            return true;
        }

        // Fast path for cleaners, cleaner 是虚引用的子类
        if (c != null) {
            c.clean();
            return true;
        }

        ReferenceQueue<? super Object> q = r.queue;
        if (q != ReferenceQueue.NULL) q.enqueue(r);
        return true;
    }
}

```

ReferenceQueue 入队后，会将Reference 加入到内置的链表中，Queue提供了poll方法用于取出元素，一般后续操作就是设置为null。例如在java.util.WeakHashMap的expungeStaleEntries方法中。

![](java_reference.webp)

### 如何工作？

Reference里有个静态字段pending，同时还通过静态代码块启动了Reference-handler thread。当一个Reference的referent被回收时，垃圾回收器会把reference添加到pending这个链表里，然后Reference-handler thread不断的读取pending中的reference，把它加入到对应的ReferenceQueue中。

可见如果pending为空的时候，会通过lock.wait()一直等在那里，其中唤醒的动作是在jvm里做的，当gc完成之后会调用如下的方法VM_GC_Operation::doit_epilogue()，在方法末尾会调用lock的notify操作，至于pending队列什么时候将引用放进去的，其实是在gc的引用处理逻辑中放进去的[<sup>3</sup>](#refer-anchor-3)。

```c
void VM_GC_Operation::doit_epilogue() {
  assert(Thread::current()->is_Java_thread(), "just checking");
  // Release the Heap_lock first.
  SharedHeap* sh = SharedHeap::heap();
  if (sh != NULL) sh->_thread_holds_heap_lock_for_gc = false;
  Heap_lock->unlock();
  release_and_notify_pending_list_lock();
}

void VM_GC_Operation::release_and_notify_pending_list_lock() {
instanceRefKlass::release_and_notify_pending_list_lock(&_pending_list_basic_lock);
}
```

我们可以通过下面代码块来进行把SoftReference，WeakReference，PhantomReference与ReferenceQueue联合使用来验证这个机制。为了确保SoftReference在每次gc后，其引用的referent都被回收，我们需要加入-XX:SoftRefLRUPolicyMSPerMB=0参数。

```java
/**
 * 为了确保System.gc()后,SoftReference引用的referent被回收需要加入下面的参数
 * -XX:SoftRefLRUPolicyMSPerMB=0
 */
public class ReferenceTest {
    private static List<Reference> roots = new ArrayList<>();

    public static void main(String[] args) throws Exception {
        ReferenceQueue rq = new ReferenceQueue();
        new Thread(new Runnable() {
            @Override
            public void run() {
                int i=0;
                while (true) {
                    try {
                        Reference r = rq.remove();
                        System.out.println("reference:"+r);
                        System.out.println( "get:"+r.get()); //为null说明referent被回收
                        i++;
                        System.out.println( "queue remove num:"+i);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();

        for(int i=0;i<100000;i++) {
            byte[] a = new byte[1024*1024];
            // 分别验证SoftReference,WeakReference,PhantomReference
            Reference r = new SoftReference(a, rq);
            //Reference r = new WeakReference(a, rq);
            //Reference r = new PhantomReference(a, rq);
            roots.add(r);
            System.gc();

            System.out.println("produce"+i);
            TimeUnit.MILLISECONDS.sleep(100);
        }
    }
}
```

通过jstack命令可以看到对应的Reference Handler thread

```java
“Reference Handler” #2 daemon prio=10 os_prio=31 tid=0x00007f8fb2836800 nid=0x2e03 in Object.wait() [0x000070000082b000]
   java.lang.Thread.State: WAITING (on object monitor)
        at java.lang.Object.wait(Native Method)
        – waiting on <0x0000000740008878> (a java.lang.ref.Reference$Lock)
        at java.lang.Object.wait(Object.java:502)
        at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
        – locked <0x0000000740008878> (a java.lang.ref.Reference$Lock)
        at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)
```

因此可以看出，当reference与referenQueue联合使用的主要作用就是当reference指向的referent回收时(或者要被回收,如下文要讲的Finalizer)，提供一种通知机制，通过queue取到这些reference，来做额外的处理工作(比 Object的finalize方法更精细[<sup>1</sup>](#refer-anchor-1))。当然，如果我们不需要这种通知机制，我们就不用传入额外的queue,默认使用NULL queue就会入队失败。

## FinalReference

FinalReference 用于处理所有的finalizer类。

```java
class FinalReference<T> extends Reference<T> {
    public FinalReference(T referent, ReferenceQueue<? super T> q) {
        super(referent, q);
    }
}
```

类访问权限是package的，这也就意味着我们不能直接去对其进行扩展，但是JDK里对此类进行了扩展实现java.lang.ref.Finalizer，这个类在概述里提到的过，而此类的访问权限也是package的，并且是final的，意味着它不能再被扩展了。

```java
final class Finalizer extends FinalReference {
    /* A native method that invokes an arbitrary object's finalize method is
       required since the finalize method is protected
     */
    static native void invokeFinalizeMethod(Object o) throws Throwable;

    private static ReferenceQueue queue = new ReferenceQueue();
    private static Finalizer unfinalized = null;
    private static final Object lock = new Object();

    private Finalizer
        next = null,
        prev = null;

    private Finalizer(Object finalizee) {
        super(finalizee, queue);
        add();
    }

    /* Invoked by VM */
    static void register(Object finalizee) {
        new Finalizer(finalizee);
    }  

    private void add() {
        synchronized (lock) {
            if (unfinalized != null) {
                this.next = unfinalized;
                unfinalized.prev = this;
            }
            unfinalized = this;
        }
    }
    // ...
   }
```

Finalizer的构造函数提供了以下几个关键信息：

* private：意味着我们无法在当前类之外构建这类的对象；
* finalizee参数：FinalReference指向的对象引用；
* 调用add方法：将当前对象插入到Finalizer对象链里，链里的对象和Finalizer类静态关联。言外之意是在这个链里的对象都无法被GC掉，除非将这种引用关系剥离（因为Finalizer类无法被unload）。

虽然外面无法创建Finalizer对象，但是它有一个名为register的静态方法，该方法可以创建这种对象，同时将这个对象加入到Finalizer对象链里，这个方法是被vm调用的，那么问题来了，vm在什么情况下会调用这个方法呢？

### 注册到Finalizer对象链 

Finalizer对象何时被注册到Finalizer对象链里?

类的修饰有很多，比如final，abstract，public等，如果某个类用final修饰，我们就说这个类是final类，上面列的都是语法层面我们可以显式指定的，在JVM里其实还会给类标记一些其他符号，比如finalizer，表示这个类是一个finalizer类（为了和java.lang.ref.Fianlizer类区分，下文在提到的finalizer类时会简称为f类），GC在处理这种类的对象时要做一些特殊的处理，如在这个对象被回收之前会调用它的finalize方法。

### 如何判断一个类是不是一个f类

在讲这个问题之前，我们先来看下java.lang.Object里的一个方法

```java
protected void finalize() throws Throwable { }
```

在Object类里定义了一个名为finalize的空方法，这意味着Java里的所有类都会继承这个方法，甚至可以覆写该方法，并且根据方法覆写原则，如果子类覆盖此方法，方法访问权限至少protected级别的，这样其子类就算没有覆写此方法也会继承此方法。

而判断当前类是否是f类的标准并不仅仅是当前类是否含有一个参数为空，返回值为void的finalize方法，还要求finalize方法必须非空，因此Object类虽然含有一个finalize方法，但它并不是f类，Object的对象在被GC回收时其实并不会调用它的finalize方法。

需要注意的是，类在加载过程中其实就已经被标记为是否为f类了。（JVM在类加载的时候会遍历当前类的所有方法，包括父类的方法，只要有一个参数为空且返回void的非空finalize方法就认为这个类是f类。）

### Finalizer.register

f类的对象何时传到Finalizer.register方法?

对象的创建其实是被拆分成多个步骤的，比如A a=new A(2)这样一条语句对应的字节码如下：

```java
0: new           #1                  // class A
3: dup
4: iconst_2
5: invokespecial #11                 // Method "<init>":(I)V
```

先执行new分配好对象空间，然后再执行invokespecial调用构造函数，JVM里其实可以让用户在这两个时机中选择一个，将当前对象传递给Finalizer.register方法来注册到Finalizer对象链里，这个选择取决于是否设置了RegisterFinalizersAtInit这个vm参数，默认值为true，也就是在构造函数返回之前调用Finalizer.register方法，如果通过-XX:-RegisterFinalizersAtInit关闭了该参数，那将在对象空间分配好之后将这个对象注册进去。

另外需要提醒的是，当我们通过clone的方式复制一个对象时，如果当前类是一个f类，那么在clone完成时将调用Finalizer.register方法进行注册。

### hotspot实现

hotspot如何实现f类对象在构造函数执行完毕后调用Finalizer.register? 

这个实现比较有意思，在这简单提一下，我们知道执行一个构造函数时，会去调用父类的构造函数，主要是为了初始化继承自父类的属性，那么任何一个对象的初始化最终都会调用到Object的空构造函数里（任何空的构造函数其实并不空，会含有三条字节码指令，如下代码所示），为了不对所有类的构造函数都埋点调用Finalizer.register方法，hotspot的实现是，在初始化Object类时将构造函数里的return指令替换为_return_register_finalizer指令，该指令并不是标准的字节码指令，是hotspot扩展的指令，这样在处理该指令时调用Finalizer.register方法，以很小的侵入性代价完美地解决了这个问题。

```java
0: aload_0
1: invokespecial #21                 // Method java/lang/Object."<init>":()V
4: return
```

#### f类对象的GC回收

在Finalizer类的clinit方法（静态块）里，我们看到它会创建一个FinalizerThread守护线程，这个线程的优先级并不是最高的，意味着在CPU很紧张的情况下其被调度的优先级可能会受到影响.

```java
private static class FinalizerThread extends Thread {
    private volatile boolean running;
    FinalizerThread(ThreadGroup g) {
        super(g, "Finalizer");
    }
    public void run() {
        if (running)
            return;
        running = true;
        for (;;) {
            try {
                Finalizer f = (Finalizer)queue.remove();
                f.runFinalizer();
            } catch (InterruptedException x) {
                continue;
            }
        }
    }
}

static {
    ThreadGroup tg = Thread.currentThread().getThreadGroup();
    for (ThreadGroup tgn = tg;
            tgn != null;
            tg = tgn, tgn = tg.getParent());
    Thread finalizer = new FinalizerThread(tg);
    finalizer.setPriority(Thread.MAX_PRIORITY - 2);
    finalizer.setDaemon(true);
    finalizer.start();
}
```

这个线程用来从queue里获取Finalizer对象，然后执行该对象的runFinalizer方法，该方法会将Finalizer对象从Finalizer对象链里剥离出来，这样意味着下次GC发生时就可以将其关联的f对象回收了，最后将这个Finalizer对象关联的f对象传给一个native方法invokeFinalizeMethod。

```java
private void runFinalizer() {
        synchronized (this) {
            if (hasBeenFinalized()) return;
            remove();
        }
        try {
            Object finalizee = this.get();
            if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
                invokeFinalizeMethod(finalizee);
                /* Clear stack slot containing this variable, to decrease
                   the chances of false retention with a conservative GC */
                finalizee = null;
            }
        } catch (Throwable x) { }
        super.clear();
    }

 static native void invokeFinalizeMethod(Object o) throws Throwable;
// invokeFinalizeMethod方法就是调了这个f对象的finalize方法
```

#### Finalizer入队

当GC发生时，GC算法会判断f类对象是不是只被Finalizer类引用（f类对象被Finalizer对象引用，然后放到Finalizer对象链里），如果这个类仅仅被Finalizer对象引用，说明这个对象在不久的将来会被回收，现在可以执行它的finalize方法了，于是会将这个Finalizer对象放到Finalizer类的ReferenceQueue里，但是这个f类对象其实并没有被回收，因为Finalizer这个类还对它们保持引用，在GC完成之前，JVM会调用ReferenceQueue中lock对象的notify方法（当ReferenceQueue为空时，FinalizerThread线程会调用ReferenceQueue的lock对象的wait方法直到被JVM唤醒），此时就会执行上面FinalizeThread线程里看到的其他逻辑了。

#### Finalizer导致的内存泄露

SocksSocketImpl的父类其实就实现了finalize方法:

```java
/**
 * Cleans up if the user forgets to close it.
 */
protected void finalize() throws IOException {
    close();
}
```

其实这么做的主要目的是万一用户忘记关闭Socket，那么在这个对象被回收时能主动关闭Socket来释放一些系统资源，但是如果用户真的忘记关闭，那这些socket对象可能因为FinalizeThread迟迟没有执行这些socket对象的finalize方法，而导致内存泄露.

#### 影响

* f对象因为Finalizer的引用而变成了一个临时的强引用，即使没有其他的强引用，还是无法立即被回收；
* f对象至少经历两次GC才能被回收，因为只有在FinalizerThread执行完了f对象的finalize方法的情况下才有可能被下次GC回收，而有可能期间已经经历过多次GC了，但是一直还没执行f对象的finalize方法。f对象成为了快速回收的阻碍者。
* CPU资源比较稀缺的情况下FinalizerThread线程有可能因为优先级比较低而延迟执行f对象的finalize方法；
* 因为f对象的finalize方法迟迟没有执行，有可能会导致大部分f对象进入到old分代，此时容易引发old分代的GC，甚至Full GC，GC暂停时间明显变长；
* f对象的finalize方法被调用后，这个对象其实还并没有被回收，虽然可能在不久的将来会被回收。
* 对于消耗非常高频的资源，不要指望finalize去承担释放资源的主要职责。推荐做法：资源用完即显式释放，或者利用资源池来复用。
* 另外，finalize会掩盖资源回收时的出错信息：

```java
// java.lang.ref.Finalizer ,Throwable被吞
private void runFinalizer(JavaLangAccess jla) {
    ...
    try {
        Object finalizee = this.get();
        if (finalizee != null && !(finalizee instanceof java.lang.Enum)) {
            jla.invokeFinalize(finalizee);
            // Clear stack slot containing this variable, to decrease
            // the chances of false retention with a conservative GC
            finalizee = null;
        }
    } catch (Throwable x) { }
    super.clear();
}
```

### `java.lang.ref.Cleaner` 代替 Finalizer

java.lang.ref.Cleaner 的实现依赖于PhantomReference(虚引用)和 ReferenceQueue。

```java
// 官方文档demo
public class CleaningExample implements AutoCloseable {
    // A cleaner, preferably one shared within a library
    private static final Cleaner cleaner = Cleaner.create();
    // 当对象变成虚引用时的action
    static class State implements Runnable {
       State() {
			System.out.println("init");// initialize State needed for cleaning action
		}
 
		public void run() {
			System.out.println("clean");// cleanup action accessing State, executed at most once
        }
    }

    private final State;
    private final Cleaner.Cleanable cleanable

    public CleaningExample() {
        this.state = new State();
        this.cleanable = cleaner.register(this, state);
    }

    public void close() {
       cleanable.clean();
    }
} 
```

仅在关联对象变为虚引用之后才调用清理操作，因此实现清理操作的对象不保留对该对象的引用非常重要。在示例中，静态内部类(静态的内部类不会持有外部类的一个隐式引用)封装了清理状态和操作，此处不使用lambda 和 内部类(匿名或非匿名),因为它们隐式包含对外部实例的引用，从而防止关联对象成为虚引用。


## 参考资料

<div id="refer-anchor-1"></div>

- [1] [Finalizers and References in Java](http://blog.ragozin.info/2016/03/finalizers-and-references-in-java.html)

- [2] [JVM 源码分析之 FinalReference 完全解读](http://www.infoq.com/cn/articles/jvm-source-code-analysis-finalreference)

<div id="refer-anchor-3"></div>

- [3] [JVM源码分析之堆外内存完全解读](http://lovestblog.cn/blog/2015/05/12/direct-buffer/)
- [4] [JDK-8138696](https://bugs.openjdk.java.net/browse/JDK-8138696)
- [5] [JDK-6534574](https://bugs.openjdk.java.net/browse/JDK-6534574)

