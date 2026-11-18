# Java虚拟机-内存分配与回收策略

对象的内存分配，主要是在堆上分配\(也可能是JIT编译后被拆散成标量类型并间接的栈上分配\):

- 对象主要分配在新生代的Eden区上，如果启动了本地线程分配缓冲，将按线程优先分配在TLAB上。
- 少数情况下会直接分配在老年代中。

分配的规则并不是百分百固定的，细节取决于当前使用的是哪一种垃圾收集器组合，还有虚拟机中与内存相关的参数设置。

对象的回收主要分为两种:

- 新生代GC\(Minor GC\)指发生在新生代的垃圾回收动作，Minor GC十分频繁，回收速度较快。  
- 老年代GC\(Major/Full GC\)指发生在老年代的GC，出现了Major GC，经常会伴随至少一次Minor GC ,但非绝对，Parallel Scavenger 收集器里有直接进行Major GC的策略选择。通常，Major GC 比Minor GC 慢10倍以上。

<!--more-->

## TLAB

内存分配的动作，可以按照线程划分在不同的空间之中进行，即每个线程在Java堆中预先分配一小块内存，称为本地线程分配缓冲（Thread Local Allocation Buffer，TLAB）。哪个线程需要分配内存，就在哪个线程的TLAB上分配。虚拟机是否使用TLAB，可以通过-XX:+/-UseTLAB参数来设定。这么做的目的之一，也是为了并发创建一个对象时，保证创建对象的线程安全性。TLAB比较小，直接在TLAB上分配内存的方式称为快速分配方式，而TLAB大小不够，导致内存被分配在Eden区的内存分配方式称为慢速分配方式。

## 对象优先在Eden分配

大多数情况，对象将优先在新生代Eden区分配。当Eden区空间不够时，将发起一次Minor GC。

```java
/**
 * -XX:+UseParNewGC
 * -Xms20m 
 * -Xmx20m 
 * -Xmn10m  
 * -XX:+PrintHeapAtGC 
 * -XX:+PrintGCDetails
 */
public class test {  
    static int mb = 1024*1024;  
      
    public static void main(String[] args) {  
        byte[] b1 = new byte[2*mb];  
        System.out.println("b1 over");  
        byte[] b2 = new byte[2*mb];  
        System.out.println("b2 over");  
        byte[] b3 = new byte[2*mb];  
        System.out.println("b3 over");//GC  
        byte[] b4 = new byte[4*mb];  
        System.out.println("b4 over");  
    }  
} 
```

堆内存大小为20M，不可自动扩展，新生代内存大小为10M，根据默认值，Eden区：Survivor区为8:1，Eden区大小应为：10M*8/10=8129KB，Survivor区大小应为1024KB，新生代总可用内存应为9216KB。

当b3分配完成后，新生代将使用6M内存（6144KB，b1+b2+b3），同时申请b4的4M=4096KB内存，此时新生代的可用内存为9216-6144=3072KB，不足以分配b4的空间，则触发一次Minor GC回收新生代内存空间，由于b1、b2以及b3都为存活状态，并且剩余的一个Survivor区无法装下b1、b2和b3，则新生代会租借老年代的区域，并将b1、b2和b3移动至租借区域，然后新生代完成Minor GC。由于此时新生代已经没有对象存放其中，剩余大量内存，则b4将在新生代中分配。

```text
b1 over  
b2 over  
b3 over  
{Heap before GC invocations=0 (full 0):  
 par new generation   total 9216K, used 6487K [0x03b30000, 0x04530000, 0x04530000)//b1+b2+b3，占6M  
  eden space 8192K,  79% used [0x03b30000, 0x04185f60, 0x04330000)  
  from space 1024K,   0% used [0x04330000, 0x04330000, 0x04430000)  
  to   space 1024K,   0% used [0x04430000, 0x04430000, 0x04530000)  
 tenured generation   total 10240K, used 0K [0x04530000, 0x04f30000, 0x04f30000)//老年代为空  
   the space 10240K,   0% used [0x04530000, 0x04530000, 0x04530200, 0x04f30000)  
 compacting perm gen  total 12288K, used 2105K [0x04f30000, 0x05b30000, 0x08f30000)  
   the space 12288K,  17% used [0x04f30000, 0x0513e478, 0x0513e600, 0x05b30000)  
No shared spaces configured.  
[GC [ParNew: 6487K->150K(9216K), 0.0092952 secs] 6487K->6294K(19456K), 0.0093314 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]//对象仍处于存活状态，新生代无足够的空间完成Minor GC，只能租借老年代的空间，将b1、b2和b3移动至老年代   
Heap after GC invocations=1 (full 0):  
 par new generation   total 9216K, used 150K [0x03b30000, 0x04530000, 0x04530000)//新生代几乎被清空  
  eden space 8192K,   0% used [0x03b30000, 0x03b30000, 0x04330000)  
  from space 1024K,  14% used [0x04430000, 0x04455a10, 0x04530000)  
  to   space 1024K,   0% used [0x04330000, 0x04330000, 0x04430000)  
 tenured generation   total 10240K, used 6144K [0x04530000, 0x04f30000, 0x04f30000)//b1+b2+b3  
   the space 10240K,  60% used [0x04530000, 0x04b30030, 0x04b30200, 0x04f30000)  
 compacting perm gen  total 12288K, used 2105K [0x04f30000, 0x05b30000, 0x08f30000)  
   the space 12288K,  17% used [0x04f30000, 0x0513e478, 0x0513e600, 0x05b30000)  
No shared spaces configured.  
}  
b4 over  
Heap  
 par new generation   total 9216K, used 4410K [0x03b30000, 0x04530000, 0x04530000)//b4  
  eden space 8192K,  54% used [0x03b30000, 0x03f82008, 0x04330000)  
  from space 1024K,  14% used [0x04430000, 0x04455a10, 0x04530000)  
  to   space 1024K,   0% used [0x04330000, 0x04330000, 0x04430000)  
 tenured generation   total 10240K, used 6144K [0x04530000, 0x04f30000, 0x04f30000)//b1+b2+b3  
   the space 10240K,  60% used [0x04530000, 0x04b30030, 0x04b30200, 0x04f30000)  
 compacting perm gen  total 12288K, used 2116K [0x04f30000, 0x05b30000, 0x08f30000)  
   the space 12288K,  17% used [0x04f30000, 0x051413c8, 0x05141400, 0x05b30000)  
No shared spaces configured.  
```

## 大对象直接进入老年代

为了避免内存回收时大对象在Eden区和2个Survivor区之间的拷贝（ParNew收集器使用复制算法），同时为了避免为了提供足够的内存空间而提前触发的GC，虚拟机提供了-XX:PretenureSizeThreshold（该设置只对Serial和ParNew收集器生效）参数，大于该参数设置值的对象将直接在老年代分配。

```java
/**
 * -XX:+UseParNewGC 
 * -Xms20m 
 * -Xmx20m 
 * -Xmn10m 
 * -XX:+PrintHeapAtGC 
 * -XX:+PrintGCDetails   
 * -XX:PretenureSizeThreshold=2097152  
 */
public class test {  
    static int mb = 1024*1024;  
      
    public static void main(String[] args) {  
        byte[] b1 = new byte[3*mb];  
        System.out.println("b1 over");  
    }  
} 
```

由于设置超过2M的对象直接在老年代分配，故b1将分配在老年代上。

```text
b1 over  
Heap  
 par new generation   total 9216K, used 507K [0x03b50000, 0x04550000, 0x04550000)//新生代几乎为空  
  eden space 8192K,   6% used [0x03b50000, 0x03bcef00, 0x04350000)  
  from space 1024K,   0% used [0x04350000, 0x04350000, 0x04450000)  
  to   space 1024K,   0% used [0x04450000, 0x04450000, 0x04550000)  
 tenured generation   total 10240K, used 3072K [0x04550000, 0x04f50000, 0x04f50000)//老年代使用了3*1024K内存  
   the space 10240K,  30% used [0x04550000, 0x04850010, 0x04850200, 0x04f50000)  
 compacting perm gen  total 12288K, used 2110K [0x04f50000, 0x05b50000, 0x08f50000)  
   the space 12288K,  17% used [0x04f50000, 0x0515f8c8, 0x0515fa00, 0x05b50000)  
No shared spaces configured.  
```

## 长期存活对象将进入老年代

由于虚拟机垃圾收集是基于“分代算法”的，故虚拟机必须能够识别哪些对象存放在新生代，哪些对象应该存放在老年代虚拟机设计了一个对象年龄计数器，如果对象在Eden区出生并且经过第一次Minor GC后依然存活，并且可以被Survivor区容纳，就会被复制至Survivor区并将对象年龄设置为1。以后对象每熬过一次Minor GC，对象年龄便+1。当对象年龄超过对象晋升老年代的年龄阀值（该阀值默认为15）时，便会晋升至老年代，何时晋升，我们接下来研究虚拟机提供了-XX：MaxTenuringThreshold参数设置晋升阀值。

```java
/**
 * -XX:+UseParNewGC 
 * -Xms20m 
 * -Xmx20m 
 * -Xmn10m 
 * -XX:+PrintHeapAtGC
 * -XX:+PrintGCDetails   
 * -XX:MaxTenuringThreshold=1  
 */
public class test {  
    static int mb = 1024*1024;  
      
    public static void main(String[] args) {  
        System.out.println("step 1");  
        byte[] b1 = new byte[1*mb/4];  
        System.out.println("step 2");  
        byte[] b2 = new byte[4*mb];  
        System.out.println("step 3");  
        byte[] b3 = new byte[4*mb];//GC  
        System.out.println("step 4");  
        b3 = null;  
        System.out.println("step 5");  
        b3 = new byte[4*mb];//GC  
    }  
}  
```

b1、b2正常分配。在step3，新生代将没有足够的内存分配b3所需的4M空间，故引发一次Minor GC。b1只有256KB，可以放置在Survivor区中，故复制b1到Survivor区中，b2为4M，无法放置到Survivor区中，故租借老年代4M内存放置b2，回收新生代内存空间，b1经历了一次Minor GC后依然存活，故年龄变为1。
在step4，分配给b3对象的内存空间依然被占用，只是将b3对象的引用置为空，由于不涉及到内存分配，故而不涉及到GC，因此对象的年龄也不会发生变化。
在step5，重新给b3对象分配4M空间，由于新生代没有足够内存，故引发Minor GC，step3分配给b3的4M内存空间由于不再与存活对象相关联，将被回收，同时，由于b1的年龄到达对象晋升老年代的年龄设置，b1将被移动至老年代。

```text
step 1  
step 2  
step 3  
{Heap before GC invocations=0 (full 0):  
 par new generation   total 9216K, used 4695K [0x03b80000, 0x04580000, 0x04580000)//b1+b2  
  eden space 8192K,  57% used [0x03b80000, 0x04015f50, 0x04380000)  
  from space 1024K,   0% used [0x04380000, 0x04380000, 0x04480000)  
  to   space 1024K,   0% used [0x04480000, 0x04480000, 0x04580000)  
 tenured generation   total 10240K, used 0K [0x04580000, 0x04f80000, 0x04f80000)//此时老年代为空  
   the space 10240K,   0% used [0x04580000, 0x04580000, 0x04580200, 0x04f80000)  
 compacting perm gen  total 12288K, used 2105K [0x04f80000, 0x05b80000, 0x08f80000)  
   the space 12288K,  17% used [0x04f80000, 0x0518e450, 0x0518e600, 0x05b80000)  
No shared spaces configured.  
[GC [ParNew: 4695K->409K(9216K), 0.0049519 secs] 4695K->4505K(19456K), 0.0049944 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]   
Heap after GC invocations=1 (full 0):  
 par new generation   total 9216K, used 409K [0x03b80000, 0x04580000, 0x04580000)//b1  
  eden space 8192K,   0% used [0x03b80000, 0x03b80000, 0x04380000)  
  from space 1024K,  39% used [0x04480000, 0x044e6610, 0x04580000)  
  to   space 1024K,   0% used [0x04380000, 0x04380000, 0x04480000)  
 tenured generation   total 10240K, used 4096K [0x04580000, 0x04f80000, 0x04f80000)//b2  
   the space 10240K,  40% used [0x04580000, 0x04980010, 0x04980200, 0x04f80000)  
 compacting perm gen  total 12288K, used 2105K [0x04f80000, 0x05b80000, 0x08f80000)  
   the space 12288K,  17% used [0x04f80000, 0x0518e450, 0x0518e600, 0x05b80000)  
No shared spaces configured.  
}  
step 4  
step 5  
{Heap before GC invocations=1 (full 0):  
 par new generation   total 9216K, used 4669K [0x03b80000, 0x04580000, 0x04580000)//b1+b3(step3)  
  eden space 8192K,  52% used [0x03b80000, 0x03fa9098, 0x04380000)  
  from space 1024K,  39% used [0x04480000, 0x044e6610, 0x04580000)  
  to   space 1024K,   0% used [0x04380000, 0x04380000, 0x04480000)  
 tenured generation   total 10240K, used 4096K [0x04580000, 0x04f80000, 0x04f80000)//b2  
   the space 10240K,  40% used [0x04580000, 0x04980010, 0x04980200, 0x04f80000)  
 compacting perm gen  total 12288K, used 2111K [0x04f80000, 0x05b80000, 0x08f80000)  
   the space 12288K,  17% used [0x04f80000, 0x0518fe08, 0x05190000, 0x05b80000)  
No shared spaces configured.  
[GC [ParNew: 4669K->43K(9216K), 0.0008256 secs] 8765K->4548K(19456K), 0.0008701 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]   
Heap after GC invocations=2 (full 0):  
 par new generation   total 9216K, used 43K [0x03b80000, 0x04580000, 0x04580000)//step3分配的b3对象空间被回收  
  eden space 8192K,   0% used [0x03b80000, 0x03b80000, 0x04380000)  
  from space 1024K,   4% used [0x04380000, 0x0438ad90, 0x04480000)  
  to   space 1024K,   0% used [0x04480000, 0x04480000, 0x04580000)  
 tenured generation   total 10240K, used 4505K [0x04580000, 0x04f80000, 0x04f80000)//b1+b2  
   the space 10240K,  43% used [0x04580000, 0x049e6590, 0x049e6600, 0x04f80000)  
 compacting perm gen  total 12288K, used 2111K [0x04f80000, 0x05b80000, 0x08f80000)  
   the space 12288K,  17% used [0x04f80000, 0x0518fe08, 0x05190000, 0x05b80000)  
No shared spaces configured.  
}  
Heap  
 par new generation   total 9216K, used 4303K [0x03b80000, 0x04580000, 0x04580000)//b3(step5)  
  eden space 8192K,  52% used [0x03b80000, 0x03fa8fe0, 0x04380000)  
  from space 1024K,   4% used [0x04380000, 0x0438ad90, 0x04480000)  
  to   space 1024K,   0% used [0x04480000, 0x04480000, 0x04580000)  
 tenured generation   total 10240K, used 4505K [0x04580000, 0x04f80000, 0x04f80000)//b1+b2  
   the space 10240K,  43% used [0x04580000, 0x049e6590, 0x049e6600, 0x04f80000)  
 compacting perm gen  total 12288K, used 2116K [0x04f80000, 0x05b80000, 0x08f80000)  
   the space 12288K,  17% used [0x04f80000, 0x051913c8, 0x05191400, 0x05b80000)  
No shared spaces configured.  
```

如果修改MaxTenuringThreshold的值为2，从打印日志中可以发现，最终老年代的内存使用量为4096KB=4M，也就是说b1没有晋升至老年代。

上面是Minor GC的运行状况，如果是Full GC呢：

```java
/**
 * -XX:+UseParNewGC 
 * -Xms20m 
 * -Xmx20m 
 * -Xmn10m 
 * -XX:+PrintHeapAtGC
 * -XX:+PrintGCDetails   
 * -XX:MaxTenuringThreshold=1
 */  
public class test {  
    static int mb = 1024*1024;  
      
    public static void main(String[] args) {  
        byte[] b1 = new byte[1*mb/4];  
        System.gc();  
    }  
}  
```

这里我们使用的是Full GC，也就是老年代的GC。Full GC通常至少伴随着一次Minor GC（并非绝对），看下面日志，这里的Minor GC应该至少发生了2次，一次Minor GC是不会把b1移动至老年代的。

```text
{Heap before GC invocations=0 (full 0):  
 par new generation   total 9216K, used 599K [0x03b80000, 0x04580000, 0x04580000)//b1  
  eden space 8192K,   7% used [0x03b80000, 0x03c15f40, 0x04380000)  
  from space 1024K,   0% used [0x04380000, 0x04380000, 0x04480000)  
  to   space 1024K,   0% used [0x04480000, 0x04480000, 0x04580000)  
 tenured generation   total 10240K, used 0K [0x04580000, 0x04f80000, 0x04f80000)//老年代为空  
   the space 10240K,   0% used [0x04580000, 0x04580000, 0x04580200, 0x04f80000)  
 compacting perm gen  total 12288K, used 2104K [0x04f80000, 0x05b80000, 0x08f80000)  
   the space 12288K,  17% used [0x04f80000, 0x0518e278, 0x0518e400, 0x05b80000)  
No shared spaces configured.  
[Full GC (System) [Tenured: 0K->404K(10240K), 0.0069434 secs] 599K->404K(19456K), [Perm : 2104K->2104K(12288K)], 0.0069992 secs] [Times: user=0.00 sys=0.00, real=0.00 secs]   
Heap after GC invocations=1 (full 1):  
 par new generation   total 9216K, used 0K [0x03b80000, 0x04580000, 0x04580000)//新生代为空  
  eden space 8192K,   0% used [0x03b80000, 0x03b80000, 0x04380000)  
  from space 1024K,   0% used [0x04380000, 0x04380000, 0x04480000)  
  to   space 1024K,   0% used [0x04480000, 0x04480000, 0x04580000)  
 tenured generation   total 10240K, used 404K [0x04580000, 0x04f80000, 0x04f80000)//b1  
   the space 10240K,   3% used [0x04580000, 0x045e5130, 0x045e5200, 0x04f80000)  
 compacting perm gen  total 12288K, used 2104K [0x04f80000, 0x05b80000, 0x08f80000)  
   the space 12288K,  17% used [0x04f80000, 0x0518e278, 0x0518e400, 0x05b80000)  
No shared spaces configured.  
}  
Heap  
 par new generation   total 9216K, used 327K [0x03b80000, 0x04580000, 0x04580000)  
  eden space 8192K,   4% used [0x03b80000, 0x03bd1f98, 0x04380000)  
  from space 1024K,   0% used [0x04380000, 0x04380000, 0x04480000)  
  to   space 1024K,   0% used [0x04480000, 0x04480000, 0x04580000)  
 tenured generation   total 10240K, used 404K [0x04580000, 0x04f80000, 0x04f80000)  
   the space 10240K,   3% used [0x04580000, 0x045e5130, 0x045e5200, 0x04f80000)  
 compacting perm gen  total 12288K, used 2116K [0x04f80000, 0x05b80000, 0x08f80000)  
   the space 12288K,  17% used [0x04f80000, 0x05191190, 0x05191200, 0x05b80000)  
No shared spaces configured.
```

## 动态对象年龄判定

为了使内存分配更加灵活，虚拟机并不要求对象年龄达到MaxTenuringThreshold才晋升老年代。如果Survivor区中相同年龄所有对象大小的总和大于Survivor区空间的一半，年龄大于或等于该年龄的对象在Minor GC时将复制至老年代。

```java
/**
 * -XX:+UseParNewGC 
 * -Xms20m 
 * -Xmx20m 
 * -Xmn10m  
 * -XX:MaxTenuringThreshold=10  
 * -XX:+PrintTenuringDistribution  
 */
public class Test {  
    static int mb = 1024*1024;  
      
    public static void main(String[] args) {  
        System.out.println("step 1");  
        byte[] b1 = new byte[1*mb/4];  
        byte[] b3 = new byte[4*mb];  
        byte[] b4 = new byte[4*mb];//GC  
        System.out.println("step 2");  
        byte[] b2 = new byte[1*mb/4];//可以尝试1*mb/2,然后观察日志  
        b4 = null;  
        System.out.println("step 3");  
        b4 = new byte[4*mb];//GC  
        System.out.println("step 4");  
        b4 = null;  
        b4 = new byte[4*mb];//GC  
    }  
}  
```

根据启动参数的设置，Survivor大小的一半是524288B，也就是512KB。第一次GC后，b1依然存活，故年龄变为1。第二次GC后，b1和b2依然存活，故b1的年龄变为2，b2的年龄为1。b1+b2的大小加起来超过了Survivor区容量的一半，此时会修改Survivor区晋升老年代年龄阀值为2（如果移动年龄为2的对象可以使Survivor去的内存使用降至512KB以内，则只移动年龄为2的对象，否则将会同时移动年龄为1的对象）。第三次GC时，将年龄等于晋升阀值的对象移动至老年代，执行GC，GC结束后，b1依然在Survivor区（当然可能从Survivor from区拷贝至了Survivor to区），此时b1的年龄变为2。这时Survivor区的使用内存没有达到512M，修改Survivor区晋升老年代年龄阀值为参数设置的10。

```text
step 1  
  
Desired survivor size 524288 bytes, new threshold 10 (max 10)  
- age   1:     412800 bytes,     412800 total  
step 2  
step 3  
  
Desired survivor size 524288 bytes, new threshold 2 (max 10)  
- age   1:     262160 bytes,     262160 total  
- age   2:     412800 bytes,     674960 total  
step 4  
  
Desired survivor size 524288 bytes, new threshold 10 (max 10)  
- age   1:        136 bytes,        136 total  
- age   2:     262160 bytes,     262296 total  
```

## 空间分配担保

由于新生代使用复制算法，当Minor GC时如果存活对象过多，无法完全放入Survivor区，就会向老年代借用内存存放对象，以完成Minor GC。

在触发Minor GC时，虚拟机会先检测之前GC时租借的老年代内存的平均大小是否大于老年代的剩余内存，
- 如果大于，则将Minor GC变为一次Full GC.
- 如果小于，则查看虚拟机是否允许担保失败,`-XX:+/-HandlePromotionFailure`。如果允许担保失败，则只执行一次Minor GC，否则也要将Minor GC变为一次Full GC(直到GC结束时才能确定到底有多少对象需要被移动至老年代，所以在GC前，只能使用粗略的平均值进行判断)。

> 从jdk6.0开始，允许担保失败已变为HotSpot虚拟机所有收集器默认设置，虚拟机将不再识别该参数设置，详见[JDK-6990095.Deprecate and eliminate -XX:-HandlePromotionFailure](https://bugs.openjdk.org/browse/JDK-6990095?page=com.atlassian.jira.plugin.system.issuetabpanels%3Aall-tabpanel)，

