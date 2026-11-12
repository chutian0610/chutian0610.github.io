# Java NIO: DirectBuffer

前面介绍过Java Buffer使用的内存分堆内内存(Heap)和堆外内存(No Heap)，本文将介绍DirectBuffer的实现原理，以DirectByteBuffer为例[^1]。

<!--more-->

## DirectByteBuffer

DirectByteBuffer是ByteBuffer的一个实现。如果需要实例化一个DirectByteBuffer，可以使用`java.nio.ByteBuffer#allocateDirect()`。

```java
public static ByteBuffer allocateDirect(int capacity) {
  return new DirectByteBuffer(capacity);
}
```

### 内存申请

先来看一下DirectByteBuffer是如何构造，如何申请与释放内存的。下面是 DirectByteBuffer 的构造器代码:

```java
DirectByteBuffer(int cap) {
    super(-1, 0, cap, cap);
    // 判断是否需要页面对齐，通过参数-XX:+PageAlignDirectMemory控制，默认为false
    boolean pa = VM.isDirectMemoryPageAligned();
    int ps = Bits.pageSize();
    // 如果不设置内存对齐，size和cap值一样
    long size = Math.max(1L, (long)cap + (pa ? ps : 0));
    //保留待分配内存
    Bits.reserveMemory(size, cap);

    long base = 0;
    try {
        // 通过unsafe.allocateMemory分配堆外内存，并返回堆外内存的基地址
        base = unsafe.allocateMemory(size);
    } catch (OutOfMemoryError x) {
        // 内存分配失败，取消预占内存
        Bits.unreserveMemory(size, cap);
        throw x;
    }
    // 初始化所有内存空间为0
    unsafe.setMemory(base, size, (byte) 0);
    if (pa && (base % ps != 0)) {
        // 页对齐
        address = base + ps - (base & (ps - 1));
    } else {
        address = base;
    }
    // 构建Cleaner对象用于跟踪DirectByteBuffer对象的垃圾回收，以实现当DirectByteBuffer被垃圾回收时，堆外内存也会被释放
    cleaner = Cleaner.create(this, new Deallocator(base, size, cap));
    att = null;
}
```

在分配内存前，尝试进行了内存预占。

```java
static void reserveMemory(long size, int cap) {
        if (!memoryLimitSet && VM.isBooted()) {
            maxMemory = VM.maxDirectMemory();
            memoryLimitSet = true;
        }
        // 乐观锁，内部逻辑是尝试循环CAS更新内存占用
        if (tryReserveMemory(size, cap)) {
            return;
        }
        final JavaLangRefAccess jlra = SharedSecrets.getJavaLangRefAccess();

        // jlra.tryHandlePendingReference()会触发一次非堵塞的Reference#tryHandlePending(false)。
        // 该方法会将已经被JVM垃圾回收的DirectBuffer对象的堆外内存释放。
        while (jlra.tryHandlePendingReference()) {
            if (tryReserveMemory(size, cap)) {
                return;
            }
        }
        // 如果已经没有足够空间，则尝试手动触发一次Full GC，触发释放堆外内存
        // 如果显式的设置-XX:+DisableExplicitGC来禁用显式GC，则System.gc()无效
        System.gc();

        // 调用System.gc()并不能够保证Full gc马上就能被执行。
        // 所以接下来while循环尝试了最大9次
        // 如果还是没有足够内存则抛出OutOfMemoryError("Direct buffer memory”)异常。 
        boolean interrupted = false;
        try {
            long sleepTime = 1;
            int sleeps = 0;
            while (true) {
                if (tryReserveMemory(size, cap)) {
                    return;
                }
                if (sleeps >= MAX_SLEEPS) {
                    break;
                }
                if (!jlra.tryHandlePendingReference()) {
                    try {
                        Thread.sleep(sleepTime);
                        sleepTime <<= 1;
                        sleeps++;
                    } catch (InterruptedException e) {
                        interrupted = true;
                    }
                }
            }

            // no luck
            throw new OutOfMemoryError("Direct buffer memory");

        } finally {
            if (interrupted) {
                // don't swallow interrupts
                Thread.currentThread().interrupt();
            }
        }
    }
```

#### 内存预占

`java.nio.Bits#tryReserveMemory` 方法尝试计算可以分配的真实内存大小，如果可以申请，则更新以下参数：

- toalCapacity: 目前用户已经申请的总空间大小；
- reservedMemory: 目前保留堆外内存总空间的大小；
- count:申请次数加1

更新时采用 CAS 更新，确保线程安全。

```java
private static boolean tryReserveMemory(long size, int cap) {
    // -XX:MaxDirectMemorySize 限制的是 cap.
    // 对于页面对齐的场景，size>=buffer
    long totalCap;
    while (cap <= maxMemory - (totalCap = totalCapacity.get())) {
        if (totalCapacity.compareAndSet(totalCap, totalCap + cap)) {
            reservedMemory.addAndGet(size);
            count.incrementAndGet();
            return true;
        }
    }

    return false;
}
```

对应于内存预占的操作是预占内存释放，也就是`java.nio.Bits#unreserveMemory`方法。

```java
static void unreserveMemory(long size, int cap) {
    long cnt = count.decrementAndGet();
    long reservedMem = reservedMemory.addAndGet(-size);
    long totalCap = totalCapacity.addAndGet(-cap);
    assert cnt >= 0 && reservedMem >= 0 && totalCap >= 0;
}
```

#### 最大直接内存

`VM.maxDirectMemory()`方法返回最大的直接内存，是由JVM属性`-XX:MaxDirectMemorySize`控制，默认是`-1`。如果没有传递该属性值的话，最大直接内存等于`Runtime#maxMemory()`。

> -XX:MaxDirectMemorySize参数只对由DirectByteBuffer分配的内存有效，对Unsafe直接分配的内存无效            

```java
// sun.misc.VM#saveAndRemoveProperties()
String s = (String)props.remove("sun.nio.MaxDirectMemorySize");
if (s != null) {
    if (s.equals("-1")) {
        // -XX:MaxDirectMemorySize not given, take default
        directMemory = Runtime.getRuntime().maxMemory();
    } else {
        long l = Long.parseLong(s);
        if (l > -1)
            directMemory = l;
    }
}
```

### 内存分配

真正分配内存的方法其实是 `unsafe.allocateMemory(size)`。这是个 Native 代码，对应的实现在openjdk的`src/hotspot/share/prims/unsafe.cpp`中:

```c
UNSAFE_LEAF(jlong, Unsafe_AllocateMemory0(JNIEnv *env, jobject unsafe, jlong size)) {
  size_t sz = (size_t)size;

  assert(is_aligned(sz, HeapWordSize), "sz not aligned");

  void* x = os::malloc(sz, mtOther);

  return addr_to_java(x);
} UNSAFE_END
```

实际上底层是调用了操作系统的malloc函数进行内存分配，然后返回一个内存地址给java。该内存地址接下来会被保存至DirectByteBuffer对象的成员变量address中。因此DirectByteBuffer本身作为一个java对象存在于jvm堆中，但是持有一个本机内存的内存地址的引用。


### 内存回收 

注意到在 DirectByteBuffer 的构造器中存在一个属性 cleaner，cleaner 实际上是一个[PhantomReference](/posts/1e5969ed/)。JVM借助 cleaner 实现了堆外内存的回收。

- cleaner和 Cleaner 直接通过双向指针关联，用于防止 cleaner 被 GC。
- cleaner 内部包含一个 Deallocator，用于在 DirectBuffer 为 null 时回收堆外内存。

![](DirectBuffer.svg)

- 当Cleaner对应的DirectBuffer被回收时，jvm 会将 Cleaner 添加到Reference.pending链表。
- ReferenceHandler 线程会不断从 pending 链表中获取 Reference。如果当前 Reference 是 Cleaner，会执行 `Cleaner.clean`方法。
- `Cleaner.clean`方法中，会先删除 Cleaner 的双向指针关联，然后执行`Deallocator.run`方法，在这个方法中会调用`unsafe.freeMemory`方法回收堆外内存。

```java
private static class Deallocator implements Runnable
{
    private static Unsafe unsafe = Unsafe.getUnsafe();
    private long address;
    private long size;
    private int capacity;
    private Deallocator(long address, long size, int capacity) {
        assert (address != 0);
        this.address = address;
        this.size = size;
        this.capacity = capacity;
    }
    public void run() {
        if (address == 0) {
            // Paranoia
            return;
        }
        unsafe.freeMemory(address);
        address = 0;
        Bits.unreserveMemory(size, capacity);
    }
}
```
### 读写操作

接下来看下 DirectByteBuffer 的读写操作。

```java
public ByteBuffer put(int i, byte x) {
    unsafe.putByte(ix(checkIndex(i)), ((x)));
    return this;
}
public byte get(int i) {
    return ((unsafe.getByte(ix(checkIndex(i)))));
}
private long ix(int i) {
    return address + ((long)i << 0);
}
```

DirectByteBuffer使用`Unsafe#getByte(long)` 和 `Unsafe#putByte(long, byte)` 这两个方法来读写堆外内存空间的指定位置的字节数据。

#### 批量操作

如果想要批量获取数据，可以使用 DirectByteBuffer 的批量方法。如果长度大于 JNI_COPY_TO_ARRAY_THRESHOLD的值，会使用JNI 方法`unsafe.copyMemory`进行分批读取。

```java
 public ByteBuffer get(byte[] dst, int offset, int length) {
  // JNI_COPY_TO_ARRAY_THRESHOLD = 6
  if (((long)length << 0) > Bits.JNI_COPY_TO_ARRAY_THRESHOLD) {
      checkBounds(offset, length, dst.length);
      int pos = position();
      int lim = limit();
      assert (pos <= lim);
      int rem = (pos <= lim ? lim - pos : 0);
      if (length > rem)
          throw new BufferUnderflowException();
          Bits.copyToArray(ix(pos), dst, arrayBaseOffset,
                            (long)offset << 0,
                            (long)length << 0);
      position(pos + length);
  } else {
      super.get(dst, offset, length);
  }
  return this;
}
static void copyToArray(long srcAddr, Object dst, long dstBaseOffset, long dstPos,
                        long length)
{
    long offset = dstBaseOffset + dstPos;
    // 分批进行(UNSAFE_COPY_THRESHOLD= 1024L * 1024L = 1M)
    while (length > 0) {
        long size = (length > UNSAFE_COPY_THRESHOLD) ? UNSAFE_COPY_THRESHOLD : length;
        unsafe.copyMemory(null, srcAddr, dst, offset, size);
        length -= size;
        srcAddr += size;
        offset += size;
    }
}
```
如果长度小于等于 JNI_COPY_TO_ARRAY_THRESHOLD，会使用父类ByteBuffer的方法:

```java
public ByteBuffer get(byte[] dst, int offset, int length) {
    checkBounds(offset, length, dst.length);
    if (length > remaining())
        throw new BufferUnderflowException();
    int end = offset + length;
    for (int i = offset; i < end; i++)
        dst[i] = get();
    return this;
}
```

对于写入也有同样的批量操作，这里就不赘述了。

### 类型兼容

DirectBuffer 支持 int，char 等类型，实际上底层都是 DirectByteBuffer。只是在读取和写入的时候每次偏移类型长度*offset。

```java
// DirectByteBuffer
// base + offset*2^0
private long ix(int i) {
    return address + ((long)i << 0);
}
// DirectCharBufferS
// base + offset*2^1
private long ix(int i) {
    return address + ((long)i << 1);
}
// DirectIntBufferS
// base + offset*2^2
private long ix(int i) {
    return address + ((long)i << 2);
}
```

[^1]: 本文JDK源码基于Java8

