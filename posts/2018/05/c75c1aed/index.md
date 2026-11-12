# Java并发之原子变量Atomic

Java从JDK1.5开始提供了java.util.concurrent.atomic包，方便程序员在多线程环境下，无锁的进行原子操作。原子变量的底层使用了处理器提供的原子指令，但是不同的CPU架构可能提供的原子指令不一样，也有可能需要某种形式的内部锁,所以该方法不能绝对保证线程不被阻塞。

<!--more-->

## Atomic 类

在Atomic包里一共有12个原子类，四种原子更新方式，分别是原子更新基本类型，原子更新数组，原子更新引用和原子更新字段。Atomic包里的类基本都是使用Unsafe实现的包装类。

### 原子更新基本类型类

用于通过原子的方式更新基本类型，Atomic包提供了以下三个类：

* AtomicBoolean：原子更新布尔类型。
* AtomicInteger：原子更新整型。
* AtomicLong：原子更新长整型。

AtomicInteger的常用方法如下：

* int addAndGet(int delta) ：以原子方式将输入的数值与实例中的值（AtomicInteger里的value）相加，并返回结果
* boolean compareAndSet(int expect, int update) ：如果输入的数值等于预期值，则以原子方式将该值设置为输入的值。
* int getAndIncrement()：以原子方式将当前值加1，注意：这里返回的是自增前的值。
* void lazySet(int newValue)：最终会设置成newValue，使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。
* int getAndSet(int newValue)：以原子方式设置为newValue的值，并返回旧值。

Atomic包提供了三种基本类型的原子更新，但是Java的基本类型里还有char，float和double等。那么问题来了，如何原子的更新其他的基本类型呢？Atomic包里的类基本都是使用Unsafe实现的，Unsafe只提供了三种CAS方法，compareAndSwapObject，compareAndSwapInt和compareAndSwapLong，再看AtomicBoolean源码，发现其是先把Boolean转换成整型，再使用compareAndSwapInt进行CAS，所以原子更新float和double也可以用类似的思路来实现(提示:可以使用 Float.floatToIntBits 和 Float.intBitstoFloat 转换来保存浮点数，使用 Double.doubleToLongBits 和 Double.longBitsToDouble 转换来保存双精度)。

#### 实现原理分析

以`AtomicInteger.incrementAndGet()`方法为例：

```java
private volatile int value;

public final int incrementAndGet() {  
    for (;;) {  
        int current = get();  
        int next = current + 1;  
        if (compareAndSet(current, next))  // CAS操作
            return next;  
    }  
}  
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);  // native function
}  
```

1. 使用了volatile 来保证多线程的可见性(cas的情况下一定只有一个线程写成功),主要是用于未内部调用unsafe类方法的原子类方法，例如`get()`。
2. 方法中采用了CAS操作，每次从内存中读取数据然后将此数据和+1后的结果进行CAS操作，如果成功就返回结果，否则重试直到成功为止。

### 原子更新数组类

通过原子的方式更新数组里的某个元素，Atomic包提供了以下三个类：

* AtomicIntegerArray：原子更新整型数组里的元素。
* AtomicLongArray：原子更新长整型数组里的元素。
* AtomicReferenceArray：原子更新引用类型数组里的元素。

AtomicIntegerArray类主要是提供原子的方式更新数组里的整型，其常用方法如下:

* int addAndGet(int i, int delta)：以原子方式将输入值与数组中索引i的元素相加。
* boolean compareAndSet(int i, int expect, int update)：如果当前值等于预期值，则以原子方式将数组位置i的元素设置成update值。

AtomicIntegerArray类需要注意的是，数组value通过构造方法传递进去，然后AtomicIntegerArray会将当前数组复制一份\(不会修改数组引用对象的值\)，所以当AtomicIntegerArray对内部的数组元素进行修改时，不会影响到传入的数组。

```java
public AtomicIntegerArray(int[] array) {
// Visibility guaranteed by final field guarantees
  this.array = array.clone();
}
```

使用例子：

```java
public class AtomicIntegerArrayTest {
  static int[] value = new int[] { 1, 2 };
  static AtomicIntegerArray ai = new AtomicIntegerArray(value);
  public static void main(String[] args) {
    ai.getAndSet(0, 3);
    System.out.println(ai.get(0));
    System.out.println(value[0]);
  }
}
// 3
// 1
```

#### 实现原理分析

以`AtomicIntegerArray.set()`方法为例：

```java
private final int[] array;
private static final int base = unsafe.arrayBaseOffset(int[].class);
static {
    int scale = unsafe.arrayIndexScale(int[].class);
    if ((scale & (scale - 1)) != 0)
        throw new Error("data type scale not a power of two");
    shift = 31 - Integer.numberOfLeadingZeros(scale);
}

 public final void set(int i, int newValue) {
        unsafe.putIntVolatile(array, checkedByteOffset(i), newValue);
} 
private long checkedByteOffset(int i) {
    if (i < 0 || i >= array.length)
        throw new IndexOutOfBoundsException("index " + i);

    return byteOffset(i);
}
private static long byteOffset(int i) {
    return ((long) i << shift) + base;
}
```

1. 使用了unsafe类来获取数组元素的偏移地址。
2. 使用了unsafe.putIntVolatile(此处的Volatile是C中的关键字，和java中的关键字并不一致)来保证操作原子性。

### 原子更新引用类型

原子更新基本类型的AtomicInteger，只能更新一个变量，如果要原子的更新多个变量，就需要使用这个原子更新引用类型提供的类。Atomic包提供了以下三个类：

* AtomicReference：原子更新引用类型。
* AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于原子的更数据和数据的版本号，可以解决使用CAS进行原子更新时，可能出现的ABA问题。
* AtomicMarkableReference：原子更新带有标记位的引用类型。可以原子的更新一个布尔类型的标记位和引用类型。AtomicStampedReference可以知道，引用变量中途被更改了几次。有时候，我们并不关心引用变量更改了几次，只是单纯的关心是否更改过，所以就有了AtomicMarkableReference。

AtomicStampedReference将版本信息和对象存储为Pair,同时更新。

```java
private static class Pair<T> {
    final T reference;
    final int stamp;
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}
private volatile Pair<V> pair;
```

使用例子:

```java
public class AtomicReferenceTest {
    public static AtomicReference<user> atomicUserRef = new AtomicReference<user>();
    public static void main(String[] args) {
        User user = new User("conan", 15);
        atomicUserRef.set(user);
        User updateUser = new User("Shinichi", 17);
        atomicUserRef.compareAndSet(user, updateUser);
        System.out.println(atomicUserRef.get().getName());
        System.out.println(atomicUserRef.get().getAge());
    }
    static class User {
        private String name;
        private int age;
        public User(String name, int age) {
            this.name = name;
            this.age = age;
        }
        public String getName() {
            return name;
        }
        public int getAge() {
            return age;
        }
    }
}
// Shinichi
// 17
```

### 原子更新字段类

如果我们只需要某个类里的某个字段，那么就需要使用原子更新字段类，Atomic包提供了以下三个类：

* AtomicIntegerFieldUpdater：原子更新整型的字段的更新器。
* AtomicLongFieldUpdater：原子更新长整型字段的更新器。
* AtomicReferenceFieldUpdater：原子更新引用类型里的字段。

> 原子更新字段类都是抽象类，每次使用都时候必须使用静态方法newUpdater创建一个更新器。原子更新类的字段的必须使用public volatile修饰符\(多线程可见\)。

AtomicIntegerFieldUpdater的例子代码如下：

```java
public class AtomicIntegerFieldUpdaterTest {
    private static AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater
            .newUpdater(User.class, "old");
    public static void main(String[] args) {
        User conan = new User("conan", 10);
        System.out.println(a.getAndIncrement(conan));
        System.out.println(a.get(conan));
    }
    public static class User {
        private String name;
        public volatile int old;
        public User(String name, int old) {
            this.name = name;
            this.old = old;
        }
        public String getName() {
            return name;
        }
        public int getOld() {
            return old;
        }
    }
}
// 10
// 11
```

#### 实现原理

```java
private static final class AtomicIntegerFieldUpdaterImpl<T> extends AtomicIntegerFieldUpdater<T> {
    private static final sun.misc.Unsafe U = sun.misc.Unsafe.getUnsafe();
    private final long offset;
    /**
      * 如果字段是protected字段, 使用子类构造更新器,否则和tclass是一致的
      */
    private final Class<?> cclass;
    /** class holding the field */
    private final Class<T> tclass;

    AtomicIntegerFieldUpdaterImpl(final Class<T> tclass, final String fieldName,final Class<?> caller) {
        final Field field;
        final int modifiers;
        try {
            field = AccessController.doPrivileged(
                new PrivilegedExceptionAction<Field>() {
                    public Field run() throws NoSuchFieldException {
                        return tclass.getDeclaredField(fieldName);
                    }
                });
            modifiers = field.getModifiers();
             sun.reflect.misc.ReflectUtil.ensureMemberAccess(caller, tclass, null, modifiers);
            ClassLoader cl = tclass.getClassLoader();
            ClassLoader ccl = caller.getClassLoader();
            if ((ccl != null) && (ccl != cl) && ((cl == null) || !isAncestor(cl, ccl))) {
                sun.reflect.misc.ReflectUtil.checkPackageAccess(tclass);
            }
        } catch (PrivilegedActionException pae) {
            throw new RuntimeException(pae.getException());
        } catch (Exception ex) {
            throw new RuntimeException(ex);
        }
        if (field.getType() != int.class)
            throw new IllegalArgumentException("Must be integer type");
        // 此处要求了被更新字段必须是volatile 修饰
        if (!Modifier.isVolatile(modifiers))
            throw new IllegalArgumentException("Must be volatile type");

            // protected 字段只允许子类或是同包类访问。
            // 如果对protected 字段的更新器来自包外，且所在class是 tclass的子类
            // 使用 caller 作为 cclass
        this.cclass = (Modifier.isProtected(modifiers) &&
                        tclass.isAssignableFrom(caller) &&
                        !isSamePackage(tclass, caller))
                        ? caller : tclass;
        this.tclass = tclass;
        this.offset = U.objectFieldOffset(field);
    }
    public final boolean compareAndSet(T obj, int expect, int update) {
        accessCheck(obj);
        return U.compareAndSwapInt(obj, offset, expect, update);
    }
    // ... ...
    
}
```

可以看到原子字段更新的原理是获取字段在对象中的偏移量,然后通过unsafe类的CAS方法进行更新。

额外提一句，在看源码的时候，你会发现一个很有意思的问题。在AtomticLong中有对平台是否支持CAS-Long的检查:

```java
/**
* Records whether the underlying JVM supports lockless
* compareAndSwap for longs. While the Unsafe.compareAndSwapLong
* method works in either case, some constructions should be
* handled at Java level to avoid locking user-visible locks.
*/
static final boolean VM_SUPPORTS_LONG_CAS = VMSupportsCS8();

/**
* Returns whether underlying JVM supports lockless CompareAndSet
* for longs. Called only once and cached in VM_SUPPORTS_LONG_CAS.
*/
private static native boolean VMSupportsCS8();
```

这个检查结果也被用在了AtomicLongFieldUpdater的构造器中:

```java
public static <U> AtomicLongFieldUpdater<U> newUpdater(Class<U> tclass,String fieldName) {
    Class<?> caller = Reflection.getCallerClass();
    if (AtomicLong.VM_SUPPORTS_LONG_CAS)
        return new CASUpdater<U>(tclass, fieldName, caller);
    else
        return new LockedUpdater<U>(tclass, fieldName, caller);
}

// casUpdate
public final boolean compareAndSet(T obj, long expect, long update) {
    accessCheck(obj);
    // U => Unsafe instance
    return U.compareAndSwapLong(obj, offset, expect, update);
}

// lockupdater
public final boolean compareAndSet(T obj, long expect, long update) { 
    accessCheck(obj);
    synchronized (this) {
        long v = U.getLong(obj, offset);
        if (v != expect)
            return false;
        U.putLong(obj, offset, update);
        return true;
    }
}
```

但是令人疑惑的是AtomLong中却是直接使用unsafe类的compareAndSwapLong方法,[为啥不一样](https://stackoverflow.com/questions/59209313/why-do-atomiclong-and-atomiclongfieldupdater-implement-compareandset-differently)?

在openJDk的MailList里我找到一些解释😂。

```mail

On 01/09/13 06:04, Aleksey Shipilev wrote:

> I actually have the question about this. What is the usual pattern for
> using AtomicLong.VM_SUPPORTS_LONG_CAS? AtomicLong seems to use Unsafe
> directly without the checks. AtomicLongFieldUpdater does the checks.
> Something is fishy about this whole thing.

Here's the story: Any implementation of Long-CAS on a machine
that does not have any other way to support it is allowed to
emulate by a synchronized block on enclosing object. For the 
AtomicXFieldUpdaters classes, there was, at the time they were
introduced, no way to express the object to use, so the checks
were done explicitly. I don't think this is even necessary
anymore, but doesn't hurt. Further, I'm not sure that JDK8
is even targeted to any machines that require this kind of
emulation.

-Doug

>> I have a concern that the Long versions of these methods may be used
>> directly without there being any check for supports_cx8
>
> I actually have the question about this. What is the usual pattern for
> using AtomicLong.VM_SUPPORTS_LONG_CAS? AtomicLong seems to use Unsafe
> directly without the checks. AtomicLongFieldUpdater does the checks.
> Something is fishy about this whole thing.

I had forgotten at what levels this operates too. As I think is now 
clear(er) there is a Java level check (and fallback to locks) for the 
AtomicLongFieldUpdater based on supports_cx8. Then there are further 
checks of supports_cx8 in unsafe.cpp. Critically in 
Unsafe_CompareAndSwapLong. (Still needed on some platforms)

Also note that Unsafe_get/setLongVolatile are also gated, unnecessarily, 
on supports_cx8. We have to have atomic 64-bit read/write for direct 
Java volatile accesses anyway. There's also an open CR to define 
supports_atomic_long_ops so that supports_cx8 is only used for the CAS, 
where it is needed, rather than simple read/write ops.

David
```

正如mail中所说,unsafe.cpp 也支持了对平台的检查。所以AtomicLong可以直接使用Unsafe的方法。

```cpp
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapLong(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jlong e, jlong x))
  UnsafeWrapper("Unsafe_CompareAndSwapLong");
  Handle p (THREAD, JNIHandles::resolve(obj));
  jlong* addr = (jlong*)(index_oop_from_field_offset_long(p(), offset));
  if (VM_Version::supports_cx8())
    return (jlong)(Atomic::cmpxchg(x, addr, e)) == e;
  else {
    jboolean success = false;
    ObjectLocker ol(p, THREAD);
    if (*addr == e) { *addr = x; success = true; }
    return success;
  }
UNSAFE_END
```

## Striped64

目前介绍的Java并发包提供的原子类都是采用volatile+CAS机制实现的，这种轻量级的实现方式比传统的synchronize一般来说更加高效，但是在高并发下依然会导致CAS操作的大量竞争失败自旋重试，这时候性能还不如使用synchronize。从JDK8开始，Java并发包新增了抽象类Striped64以及它的扩展类 LongAdder、LongAccumulator、DoubleAdder、DoubleAccumulator解决了高并发下的累加问题。

Striped64的原理很简单，Striped64不再使用单个变量保存结果，而是包含一个基础值base和一个单元哈希表cells(其实就是一个数组)。没有竞争的情况下，要累加的数会累加到这个基础值上；如果有竞争的话，会通过内部的分散计算将要累加的数累加到单元哈希表中的某个单元里面。所以整个Striped64的值包括基础值和单元哈希表中所有单元的值的总和。显然Striped64是一种以空间换时间的解决方案。

```java
abstract class Striped64 extends Number {
    //存放Cell的哈希表，大小为2的幂
    transient volatile Cell[] cells;

    //基础值, 主要时当没有竞争是直接更新这个值, 但也可以作为哈希表初始化竞争失败的回退方案
    //通过CAS的方式更新
    transient volatile long base;

    //自旋锁（通过CAS方式），用于当需要扩展数组的容量或创建一个数组中的元素时加锁.
    transient volatile int cellsBusy;

    /**
     * CASes the base field.
     */
    final boolean casBase(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, BASE, cmp, val);
    }

    /**
     * CASes the cellsBusy field from 0 to 1 to acquire lock.
     */
    final boolean casCellsBusy() {
        return UNSAFE.compareAndSwapInt(this, CELLSBUSY, 0, 1);
    }
    ... ...
}
```

### cell

cell 代表一次针对 base 的变更。

```java
@sun.misc.Contended static final class Cell {
    volatile long value;
    Cell(long x) { value = x; }
    final boolean cas(long cmp, long val) {
        return UNSAFE.compareAndSwapLong(this, valueOffset, cmp, val);
    }

    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    private static final long valueOffset;
    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> ak = Cell.class;
            valueOffset = UNSAFE.objectFieldOffset
                (ak.getDeclaredField("value"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

### 累加逻辑

Striped64的累加逻辑如下:

```java
// wasUncontended 表示某次更新 cell 的过程中 CAS 失败。
final void longAccumulate(long x, LongBinaryOperator fn,
                              boolean wasUncontended) {
    int h; 
    if ((h = getProbe()) == 0) { //获取当前线程的probe值作为hash值。  
        ThreadLocalRandom.current(); //如果probe值为0，强制初始化当前线程的probe值
        h = getProbe();
        wasUncontended = true; //重新计算了hash值之后，将未竞争标识为true  
    }
    boolean collide = false;         // True if last slot nonempty
    for (;;) {
        Cell[] as; Cell a; int n; long v;
        if ((as = cells) != null && (n = as.length) > 0) { //哈希表已经初始化过了
            //通过当前线程的probe值来定位当前线程被分散到的Cell数组中的Slot 
            if ((a = as[(n - 1) & h]) == null) {
                // 如果自旋锁标记为空闲  
                if (cellsBusy == 0) {       // Try to attach new Cell
                    Cell r = new Cell(x);   // Optimistically create
                    if (cellsBusy == 0 && casCellsBusy()) {
                        // 通过 CAS 方式，向 Slot 中加入新 CELL
                        boolean created = false;
                        try {               // Recheck under lock
                            Cell[] rs; int m, j;
                            if ((rs = cells) != null &&
                                (m = rs.length) > 0 &&
                                rs[j = (m - 1) & h] == null) {
                                rs[j] = r;
                                created = true;
                            }
                        } finally {
                            cellsBusy = 0; //释放cellsBusy锁。  
                        }
                        if (created)
                            break; //如果创建成功，直接跳出循环，退出方法。
                        continue;  //说明上面指定的Slot有cell了，继续尝试。  
                    }
                }
                collide = false;
            }
            // 此时说明 Slot 已经有值了。
            else if (!wasUncontended)       // CAS already known to fail
                // 如果上次 Slot CAS更新值失败，重置标志位。
                // 等待 Thread advanceProbe后重试
                wasUncontended = true;      // Continue after rehash
            else if (a.cas(v = a.value, ((fn == null) ? v + x :
                                            fn.applyAsLong(v, x))))
                //如果还未发生竞争，则尝试将x累加到该slot上，成功则返回
                break;
            else if (n >= NCPU || cells != as) //到这里说明该位置不为空，但是尝试累加到该位置上失败
                //如果哈希表即数组cells已经到最大或数组发生了变化  
49              //这里设置冲突collide为false，等待 Thread advanceProbe后重试。
                collide = false;            // At max size or stale
            else if (!collide)
                collide = true; //设置冲突标志，表示发生了冲突，重试。
            else if (cellsBusy == 0 && casCellsBusy()) {
            //到这里说明该位置不为空，但是尝试累加到该位置上失败，并且数组的容量还未到最大值，数组也没有发生变化，但是发生了冲突
                try {
                    if (cells == as) {      // Expand table unless stale
                        //对数组进行扩容,容量乘2
                        Cell[] rs = new Cell[n << 1];
                        for (int i = 0; i < n; ++i)
                            rs[i] = as[i];
                        cells = rs;
                    }
                } finally {
                    cellsBusy = 0; // 释放锁
                }
                collide = false;
                continue;        //扩容哈希表后，然后重试。      
            }
            //重新计算新的probe值以对应到不同的下标元素，然后重试。 
            h = advanceProbe(h);
        }
        else if (cellsBusy == 0 && cells == as && casCellsBusy()) {
            //哈希表还未创建，尝试获取cellsBusy锁，成功  
            boolean init = false;
            try {                           // Initialize table
                if (cells == as) {
                    //初始化哈希表cells，初始容量为2  
                    Cell[] rs = new Cell[2];
                    rs[h & 1] = new Cell(x);
                    cells = rs;
                    init = true;
                }
            } finally {
                cellsBusy = 0; //释放cellsBusy锁  
            }
            if (init)
                break;
        }
        //如果创建哈希表由于竞争导致失败，尝试将x累加到base上。  
        else if (casBase(v = base, ((fn == null) ? v + x :
                                    fn.applyAsLong(v, x))))
            break; //成功累加到base上，退出方法，结束       
    }
}
```

代码逻辑如下:

- if 该哈希表即数组已经初始化过了
    -  if 映射到的槽位（下标）是空的，即还没有放置过元素
        - if 锁空闲，加锁后再次判断，如果该槽位仍然是空的，初始化cell并放到该槽。成功后退出。
        - 锁已经被占用了，设置collide为false，会导致重新产生哈希重试。
    - else if （槽不为空）在槽上之前的CAS已经失败，刷新哈希重试。
    - else if （槽不为空、且之前的CAS没失败）在此槽的cell上尝试更新，成功退出。
    - else if 表已达到容量上限或被扩容了，刷新哈希重试。
    - else if 如果不存在冲突，则设置为存在冲突，刷新哈希重试。
    - else if 如果成功获取到锁，则扩容。
    - else 刷新哈希值，尝试其他槽。
- else if （表未初始化）锁空闲，且数组无变化，且成功获取到锁：
    - 初始化哈希表的大小为2，根据取模（h & 1）将需累加的参数x放到对应的下标中。释放锁。
-  else if （表未初始化，锁不空）尝试直接在base上更新，成功返回，失败回到步骤1重试。


Striped64对性能的提升原因重要的点有以下两点：

1. 加锁的时机：只有在初始化表、扩展表空间和在空槽位上放入值的时候才会加锁，其它时候都采用乐观锁CAS直接尝试，失败之后通过advanceProbe方法刷新哈希值之后换到不同的槽位继续尝试，而不是死等。这种方式在高并发下显然是高效的。
2. 数组的容量：初始容量为2，以后每次通过移位运算增加容量，保证容量大小是2的幂，所以可以使用(length - 1) & h这种取模方式来索引，容量大小上限为大于等于CPU核心数，这是因为如果每个线程对应一个CPU核心，将会存在一个完美的哈希函数映射线程到不同的槽位（数组下标），从而可以尽可能的消除多个线程竞争同一个槽位，提高了并发效率。

### LongAdder.add

```java
public void add(long x) {
    Cell[] as; long b, v; int m; Cell a;
    //会在哈希表未初始化的时候才尝试在base上累加
    if ((as = cells) != null || !casBase(b = base, b + x)) {
        // 如果已经初始化了哈希表（但对应的槽位为空或尝试累加到该槽位失败）
        // 或者在base上累加失败， 则调用父类Striped64的longAccumulate方法进行分散累加。
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended = a.cas(v = a.value, v + x)))
            longAccumulate(x, null, uncontended);
    }
}
```

### LongAccumulator.accumulate

LongAccumulator和DoubleAccumulator的构造方法与LongAdder/DoubleAdder不同，LongAdder/DoubleAdder只有一个无参的构造方法，不能指定初始值，而它们的构造方法有两个参数，第一个参数是一个需要被实现累加逻辑的函数接口，第二个参数就是初始值。

> 线程之间的累加顺序无法保证，也不应该被依赖，它们仅仅适用于对累加顺序不敏感的累加操作，构造方法的第一个参数指定的累加函数必须是无副作用的，例如（x*2+y）这样的累加函数就不适用在这里，

```java
public void accumulate(long x) {
    Cell[] as; long b, v, r; int m; Cell a;
    if ((as = cells) != null ||
        (r = function.applyAsLong(b = base, x)) != b && !casBase(b, r)) {
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[getProbe() & m]) == null ||
            !(uncontended =
                (r = function.applyAsLong(v = a.value, x)) == v ||
                a.cas(v, r)))
            longAccumulate(x, function, uncontended);
    }
}
```

