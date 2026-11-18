# Java并发之ThreadLocal

ThreadLocal类提供的以下几个方法：

```java
// 用来获取ThreadLocal在当前线程中保存的变量副本
public T get()
// 用来设置当前线程中变量的副本
public void set(T value)
// 用来移除当前线程中变量的副本
public void remove()
// 用来在使用时进行重写的
protected T initialValue()
```

<!--more-->

## 深入源码

我们先看`ThreadLocal.get()`方法，方法代码如下：

```java
public T get() {
    // 取得当前线程
    Thread t = Thread.currentThread();
    // 通过getMap(t)方法获取到一个ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 获取到<key,value>键值对，注意这里获取键值对传进去的是this(ThreadLocal)，而不是当前线程thread
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            // 如果获取成功，则返回value值。
            T result = (T)e.value;
            return result;
       }
    }
    //如果map为空，则调用setInitialValue方法返回value。
    return setInitialValue();
}
```
再看下ThreadLocalMap中的 getMap 方法。
```java
ThreadLocalMap getMap(Thread t) {
    // 在getMap中，调用当期线程t，返回当前线程t中的一个成员变量threadLocals。
    // Thread类中成员变量threadLocals是一个ThreadLocalMap 实例
    // ThreadLocalMap 是ThreadLocal类的一个内部类
    return t.threadLocals;
}
// Class ThreadLocalMap
private Entry getEntry(ThreadLocal<?> key) {
    // 此处代码分析见下方HASH 碰撞
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    // 检查指定位置
    if (e != null && e.get() == key)
        return e;
    else
        // 没有在指定位置找到值,可能是被拉链法，放到了后面的槽中。
        return getEntryAfterMiss(key, i, e);
}
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len); // 遍历hash table
        e = tab[i];
    }
    return null;
}
```

然后再继续看setInitialValue方法的具体实现：

```java
private T setInitialValue() {
    // 懒加载initialValue()
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### 总结一下

* 首先，在每个线程Thread内部有一个ThreadLocal.ThreadLocalMap类型的成员变量threadLocals，这个threadLocals就是用来存储实际的变量副本的，键值为当前ThreadLocal变量，value为变量副本（即T类型的变量）。

* 初始时，在Thread里面，threadLocals为空，当通过ThreadLocal变量调用get()方法或者set()方法，就会对Thread类中的threadLocals进行初始化，并且以当前ThreadLocal变量为键值，以ThreadLocal要保存的副本变量为value，存到threadLocals。

* 然后在当前线程里面，如果要使用副本变量，就可以通过get方法在threadLocals里面查找。在进行get之前，必须先set，否则会报空指针异常；如果想在get之前不需要调用set就能正常访问的话，必须重写initialValue()方法。因为在上面的代码分析过程中，我们发现如果没有先set的话，即在map中查找不到对应的存储，则会通过调用setInitialValue方法返回i，而在setInitialValue方法中，有一个语句是T value = initialValue()， 而默认情况下，initialValue方法返回的是null。

## hash 碰撞

既然ThreadLocal用map就避免不了冲突的产生，这里碰撞其实有两种类型:

1. 只有一个ThreadLocal实例的时候，当向thread-local变量中设置多个值的时产生的碰撞。
2. 多个ThreadLocal实例的时候，最极端的是每个线程都new一个ThreadLocal实例，此时利用特殊的哈希码`0x61c88647`大大降低碰撞的几率， 同时利用开放定址法处理碰撞.

```java
// Class ThreadLocalMap
private void set(ThreadLocal<?> key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    // 开放定址，不断寻找新的hash地址
    for (Entry e = tab[i];e != null;e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();
        // 同一个ThreadLocal实例,覆盖值
        if (k == key) {
            e.value = value;
            return;
        }
        // 找到key空的hash地址
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }
    // 找不到开放的地址
    tab[i] = new Entry(key, value);
    int sz = ++size; // 新增元素
    // 回收不成功且空间不足,扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
private static int nextIndex(int i, int len) {
    return ((i + 1 < len) ? i + 1 : 0); // 循环遍历hash table
}
//Class ThreadLocal
private final int threadLocalHashCode = nextHashCode();
private static AtomicInteger nextHashCode =new AtomicInteger();
/**
 * The difference between successively generated hash codes - turns
 * implicit sequential thread-local IDs into near-optimally spread
 * multiplicative hash values for power-of-two-sized tables.
 */
private static final int HASH_INCREMENT = 0x61c88647;
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

在上面的代码中你可以看到一个魔数`0x61c88647`,作用在注释中已经写的很清楚了：为了让哈希码能均匀的分布在2的N次方的数组里。来看一下ThreadLocal怎么使用的这个 threadLocalHashCode 哈希码的。

在上面的set方法中用`key.threadLocalHashCode & (len-1)`来定位。注意，ThreadLocalMap 中 Entry[] table 的大小必须是2的N次方.那么`len-1`的二进制表示就是低位连续的N个1, key.threadLocalHashCode & (len-1) 的值就是 threadLocalHashCode 的低N位, 

```java
/**
 * The table, resized as necessary.
 * table.length MUST always be a power of two.
 */
private Entry[] table;
```

我们通过代码测试一下，0x61c88647 是否能让哈希码能均匀的分布在2的N次方的数组里。

```java
public class MagicHashCode {
    private static final int HASH_INCREMENT = 0x61c88647;

    public static void main(String[] args) {
        hashCode(16); //初始化16
        hashCode(32); //后续2倍扩容
    }

    private static void hashCode(Integer length){
        int hashCode = 0;
        for(int i=0; i< length; i++){
            hashCode = i * HASH_INCREMENT+HASH_INCREMENT;//每次递增HASH_INCREMENT
            System.out.print(hashCode & (length-1));
            System.out.print(" ");
        }
        System.out.println();
    }
}
// 产生的哈希码分布确实是很均匀，而且没有任何冲突
// 7 14 5 12 3 10 1 8 15 6 13 4 11 2 9 0 
// 7 14 21 28 3 10 17 24 31 6 13 20 27 2 9 16 23 30 5 12 19 26 1 8 15 22 29 4 11 18 25 0 
```

再看下面一段代码：

```java
public class ThreadHashTest {
    public static void main(String[] args) {
        long l1 = (long) ((1L << 32) * (Math.sqrt(5) - 1)/2);
        System.out.println("as 32 bit unsigned: " + l1);
        int i1 = (int) l1;
        System.out.println("as 32 bit signed:   " + i1);
        System.out.println("MAGIC = " + 0x61c88647);
    }
}
// 结果：
// as 32 bit unsigned: 2654435769
// as 32 bit signed:   -1640531527
// MAGIC = 1640531527
```

`0x61c88647` 与一个神奇的数字产生了关系，它就是 `(Math.sqrt(5) - 1)/2`。也就是传说中的黄金比例0.618，即 `0x61c88647 = 2^32 * 黄金分割比`。

## ThreadLocalMap的内存优化

下面继续看ThreadLocalMap的实现,可以看到ThreadLocalMap的Entry继承了WeakReference，并且使用ThreadLocal作为键值。

```java
static class ThreadLocalMap {
    static class Entry extends WeakReference<ThreadLocal<?>> {
        Object value;
        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }
```

Entry是ThreadLocal的弱引用,那么会出现key为null的情况。这表示threadLocal实例已被回收,那么就会出现hash table 槽不为空，但是key为空的情况。这些值是无意义的，需要回收处理。ThreadLocal在`get()`,`set()`和`remove()`方法的执行中都做了优化，会清除ThreadLocalMap中所有key为null的 value。

* `get()`和`remove()` -> `expungeStaleEntry(int staleSlot)`
* `set()` -> `replaceStaleEntry(ThreadLocal<?> key, Object value,int staleSlot)`

但是这些被动措施不能保证不会内存泄漏。

```java
private void replaceStaleEntry(ThreadLocal<?> key, Object value,
                                int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;
    Entry e;

    // 向前检查key为null的槽,
    int slotToExpunge = staleSlot;
    for (int i = prevIndex(staleSlot, len);
            (e = tab[i]) != null; // 直到有空槽为止
            i = prevIndex(i, len))
        if (e.get() == null)
            slotToExpunge = i;
    
    // 向后检查key为null的槽
    for (int i = nextIndex(staleSlot, len);
            (e = tab[i]) != null; // 直到有空槽为止
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();

        // 如皋找到key对应的位置，需要交换他们的值，来维护哈希表顺序(开发地址法)
        if (k == key) {
            e.value = value; // 更新value
            // 交换位置
            tab[i] = tab[staleSlot];
            tab[staleSlot] = e; 

            // 删除
            if (slotToExpunge == staleSlot)
                slotToExpunge = i;
            cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
            return;
        }

        // 未能找到key
        if (k == null && slotToExpunge == staleSlot)
            slotToExpunge = i;
    }

    // If key not found, put new entry in stale slot
    tab[staleSlot].value = null;
    tab[staleSlot] = new Entry(key, value);

    // If there are any other stale entries in run, expunge them
    if (slotToExpunge != staleSlot)
        cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);
}
// 删除staleSlot 位置的enrty,并优化后面相连的槽
// 由于开发定址法.整个hash table被分为 空槽隔开的若干槽(由于地址冲突相连)
private int expungeStaleEntry(int staleSlot) {
    Entry[] tab = table;
    int len = tab.length;

    // 删除 entry
    tab[staleSlot].value = null;
    tab[staleSlot] = null;
    size--;

    // 重建部分hash 表
    Entry e;
    int i;
    for (i = nextIndex(staleSlot, len);
            (e = tab[i]) != null;
            i = nextIndex(i, len)) {
        ThreadLocal<?> k = e.get();
        if (k == null) {
            e.value = null;
            tab[i] = null;
            size--;
        } else {
            int h = k.threadLocalHashCode & (len - 1);
            if (h != i) {
                tab[i] = null;
                while (tab[h] != null)
                    h = nextIndex(h, len);
                tab[h] = e;
            }
        }
    }
    return i;
}
// 优化整个hash table的slots
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```

## 内存泄漏问题

ThreadLocalMap使用ThreadLocal的弱引用作为Key，如果一个ThreadLocal没有外部强引用时，那么系统GC时，会回收这个ThreadLocal。这样一来ThreadLocalMap中就会出现key为null的Entry，就无法访问这些key为null的entry的value，如果当前线程一直不结束，这些key为null的entry的value就会一直存在一条强引用：Thread -> ThreadLocalMap -> Entry -> value 从而导致内存泄漏。

### static 

其次，ThreadLocal一般会采用static修饰。这样做既有好处，也有坏处。好处是它一定程度上可以避免错误，至少它可以避免重复创建TSO（Thread Specific Object，即ThreadLocal所关联的对象）所导致的浪费。坏处是这样做可能正好形成内存泄漏所需的条件。

我们知道，一个ThreadLocal实例对应当前线程中的一个TSO实例。因此，如果把ThreadLocal声明为某个类的实例变量（而不是静态变量），那么每创建一个该类的实例就会导致一个新的TSO实例被创建。显然，这些被创建的TSO实例是同一个类的实例。于是，同一个线程可能会访问到同一个TSO（指类）的不同实例，这即便不会导致错误，也会导致浪费（重复创建等同的对象）！因此，一般我们将ThreadLocal使用static修饰即可。

在 Tomcat 中，下面的代码都在 webapp 内，会导致WebappClassLoader泄漏，无法被回收。

```java
public class MyCounter {
        private int count = 0;

        public void increment() {
                count++;
        }

        public int getCount() {
                return count;
        }
}

public class MyThreadLocal extends ThreadLocal<MyCounter> {
}

public class LeakingServlet extends HttpServlet {
        private static MyThreadLocal myThreadLocal = new MyThreadLocal();

        protected void doGet(HttpServletRequest request,
                        HttpServletResponse response) throws ServletException, IOException {

                MyCounter counter = myThreadLocal.get();
                if (counter == null) {
                        counter = new MyCounter();
                        myThreadLocal.set(counter);
                }

                response.getWriter().println(
                                "The current thread served this servlet " + counter.getCount()
                                                + " times");
                counter.increment();
        }
}
```

上面的代码中，只要LeakingServlet被调用过一次，且执行它的线程没有停止，就会导致WebappClassLoader泄漏。如果tomcat reload 多次，WebappClassLoader就会加载多次，它加载的类都无法被卸载，最后导致 PermGen OutOfMemoryException。

对于运行在 Java EE容器中的 Web 应用来说，类加载器的实现方式与一般的 Java 应用有所不同。不同的 Web 容器的实现方式也会有所不同。以 Apache Tomcat 来说，每个 Web 应用都有一个对应的类加载器实例。该类加载器也使用代理模式，所不同的是它是首先尝试去加载某个类，如果找不到再代理给父类加载器。这与一般类加载器的顺序是相反的。这是 Java Servlet 规范中的推荐做法，其目的是使得 Web 应用自己的类的优先级高于 Web 容器提供的类。这种代理模式的一个例外是：Java 核心库的类是不在查找范围之内的。这也是为了保证 Java 核心库的类型安全。

* LeakingServlet持有static的MyThreadLocal，导致myThreadLocal的生命周期跟LeakingServlet类的生命周期一样长。意味着myThreadLocal不会被回收，弱引用形同虚设，所以当前线程无法通过ThreadLocalMap的防护措施清除counter的强引用;
* 强引用链：thread -> threadLocalMap -> counter -> MyCounter.class -> WebappClassLocader，导致WebappClassLoader泄漏;

**最好的方法是每次使用完 ThreadLocal 都调用`remove()`方法清除数据**。

## InheritableThreadLocal

使用InheritableThreadLocal可以让具有继承关系的子线程从父线程中取得值。子类还可以继承父类的值再对父类的值进行修改，只需要在InheritableThreadLocal的实例中重写childValue( )方法，设置子线程的值就可以了。

```java
// Thread类中存在两个变量：
ThreadLocal.ThreadLocalMap threadLocals = null;
// 给子线程使用
ThreadLocal.ThreadLocalMap inheritableThreadLocals = null;
```

在Thread的构造器中有继承父线程的inheritableThreadLocals的代码:

```java
// Thread() -> init()
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
        // ... ...
        Thread parent = currentThread();
        // ... ...
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        tid = nextThreadID();
    }
// ThreadLocal
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
    return new ThreadLocalMap(parentMap);
}
private ThreadLocalMap(ThreadLocalMap parentMap) {
    Entry[] parentTable = parentMap.table;
    int len = parentTable.length;
    setThreshold(len);
    table = new Entry[len];
    // 逐一复制 parentMap 的记录
    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            @SuppressWarnings("unchecked")
            ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
            if (key != null) {
                // childValue 被 InheritableThreadLocal覆盖
                Object value = key.childValue(e.value); 
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```

再看InheritableThreadLocal类的源码：

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
 protected T childValue(T parentValue) {
        return parentValue;
    }
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

子线程可以通过修改可变性（Mutable）对象对主线程才是可见的，即才能将修改传递给主线程，但这不是一种好的实践，不建议使用，为了保护线程的安全性，一般建议只传递不可变（Immuable）对象，即没有状态的对象。如果想在修改子线程可变对象，同时不影响主线程，可以通过重写childValue()方法来实现。在childValue()中返回对象的拷贝。


生产中使用可以参考阿里的[transmittable thread local](https://github.com/alibaba/transmittable-thread-local)


## 参考

- [1] [tomcat wiki.MemoryLeakProtection](https://wiki.apache.org/tomcat/MemoryLeakProtection)

