# Java虚拟机-对象内存布局

本文讨论的是Java 对象的内存布局, 默认使用Oracle 1.8 Hotspot虚拟机。

<!--more-->

Java对象的内存布局：对象头（Header），实例数据（Instance Data）和对齐填充（Padding）。

```text
+------------------+
|      Header      |
+------------------+
|   Instance Data  |
+------------------+
|      Padding     |
+------------------+
```

对象大小分为:

* 自身的大小（Shadow heap size）
* 所直接或间接引用的对象的大小（Retained heap size）,一般可以通过遍历引用的方式得到。


## Header

```text
+------------------+------------------+-------------------+
|    mark word     |   klass pointer  |  array size (opt) |
+------------------+------------------+-------------------+
```

上面是对象头的结构，结构说明如下：

|结构块|描述|32位vm|64位vm|64位指针压缩|
|:---|:---|:---|:---|:---|
|mark word|锁信息(Synchronization)，GC信息和对象hashCode|32bit|64bit|64bit|
|klass pointer|类型指针(指向对象的Class对象的内存地址)|32bit|64bit|32bit|
|array size|如果是对象是数组才会有这个部分，对象头部有一个保存数组长度的空间|32bit|64bit|32bit|

* 32位JVM的对象头

```text
|----------------------------------------------------------------------------------------|--------------------|
|                                    Object Header (64 bits)                             |        State       |
|-------------------------------------------------------|--------------------------------|--------------------|
|                  Mark Word (32 bits)                  |      Klass pointer (32 bits)   |                    |
|-------------------------------------------------------|--------------------------------|--------------------|
| identity_hashcode:25 | age:4 | biased_lock:1 | lock:2 |      OOP to metadata object    |       Normal       |
|-------------------------------------------------------|--------------------------------|--------------------|
|  thread:23 | epoch:2 | age:4 | biased_lock:1 | lock:2 |      OOP to metadata object    |       Biased       |
|-------------------------------------------------------|--------------------------------|--------------------|
|               ptr_to_lock_record:30          | lock:2 |      OOP to metadata object    | Lightweight Locked |
|-------------------------------------------------------|--------------------------------|--------------------|
|               ptr_to_heavyweight_monitor:30  | lock:2 |      OOP to metadata object    | Heavyweight Locked |
|-------------------------------------------------------|--------------------------------|--------------------|
|                                              | lock:2 |      OOP to metadata object    |    Marked for GC   |
|-------------------------------------------------------|--------------------------------|--------------------|
```

* 64位JVM的对象头

```text
|------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (128 bits)                                        |        State       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                         |    Klass pointer (64 bits)  |                    |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | unused:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                       ptr_to_lock_record:62                         | lock:2 |    OOP to metadata object   | Lightweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor:62                   | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                     | lock:2 |    OOP to metadata object   |    Marked for GC   |
|------------------------------------------------------------------------------|-----------------------------|--------------------|
```

* 64位开启压缩的JVM对象头

```text
|--------------------------------------------------------------------------------------------------------------|--------------------|
|                                            Object Header (96 bits)                                           |        State       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                  Mark Word (64 bits)                           |    Klass pointer (32 bits)  |                    |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
| unused:25 | identity_hashcode:31 | cms_free:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Normal       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
| thread:54 |       epoch:2        | cms_free:1 | age:4 | biased_lock:1 | lock:2 |    OOP to metadata object   |       Biased       |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                         ptr_to_lock_record                            | lock:2 |    OOP to metadata object   | Lightweight Locked |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                     ptr_to_heavyweight_monitor                        | lock:2 |    OOP to metadata object   | Heavyweight Locked |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
|                                                                       | lock:2 |    OOP to metadata object   |    Marked for GC   |
|--------------------------------------------------------------------------------|-----------------------------|--------------------|
```

|字段|说明|
|:---|:---|
|hashcode|保存对象的哈希码|
|age|保存对象的分代年龄|
|biased_lock|偏向锁标识位|
|lock|锁状态标识位|
|thread|保存持有偏向锁的线程ID|
|epoch|保存偏向时间戳|
|ptr_to_lock_record|指向栈中锁记录的指针|
|ptr_to_heavyweight_monitor|指向重量级指针|

MarkWord（标记字段） ：哈希码、分代年龄、锁标志位、偏向线程ID、偏向时间戳等信息。Mark Word被设计成了一个非固定的数据结构以便在极小的空间内存储尽量多的信息，它会根据对象的状态(lock)区分不同的状态位，从而区分不同的存储结构。

Klass Pointer（类型指针）： 即指向当前对象的类的元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例。并不是所有的虚拟机实现都必须在对象数据上保留类型指针，换句话说查找对象的元数据信息并不一定要经过对象本身。

另外，如果是数组，**对象头中还有一块用于存放数组长度的数据**，因为虚拟机可以通过普通Java对象类的元数据信息确定Java对象的大小，但是从数组类的元数据中无法确定数组的大小。

## Instance Data

对象实例数据包括了对象的所有成员变量，其大小由各个成员变量的大小决定，成员变量的大小一般是基于变量所属类型。

|type|size(bits)|bytes|
|:---|:---|:---|
|boolean|8|1|
|byte|8|1|
|char|16|2|
|short|16|2|
|int|32|4|
|long|64|8|
|float|32|4|
|double|64|8|

在 32 位的 JVM 上，一个对象引用占用 4 个字节；在 64 位上，占用 8 个字节。

可能认为布尔值占用一位或一个字节的八分之一。在虚拟机的具体实现中，可以是1位，1字节，或者最有可能的是，它占用基础硬件的基本字长（32或64位）。
> [Hotspot](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.3.4)中每个boolean占用1个字节。

注意，非静态的内部类，有一个隐藏的对外部类的引用。

### 字段重排序

JVM 中为了减少内存空隙，会对类中的字段进行重排。JVM 中对齐字段的顺序是 8->4->2->1，就是先放大的字段，再放小的字段。当8字节对齐的字段压紧后留下的空隙可以使用一个或几个更小的字段来填充。

```java
class MyClass {
    byte a;
    int c;
    boolean d;
    long e;
    Object f;        
}
```

内存布局(field重排序)如下：

```text
32 bit                      64bit +UseCompressedOops

[HEADER:  8 bytes]  8           [HEADER: 12 bytes] 12
[e:       8 bytes] 16           [e:       8 bytes] 20
[c:       4 bytes] 20           [c:       4 bytes] 24
[a:       1 byte ] 21           [a:       1 byte ] 25
[d:       1 byte ] 22           [d:       1 byte ] 26
[padding: 2 bytes] 24           [padding: 2 bytes] 28
[f:       4 bytes] 28           [f:       4 bytes] 32
[padding: 4 bytes] 32
```

### 父类和子类的实例成员

继承关系中，JVM会首先存放超类的字段。

```java
class A {
    byte a;
}

class B extends A {
    byte b;
}
```

内存布局如下：

```text
32 bit                  64bit +UseCompressedOops

[HEADER:  8 bytes]  8       [HEADER: 12 bytes] 12
[a:       1 byte ]  9       [a:       1 byte ] 13
[padding: 3 bytes] 12       [padding: 3 bytes] 16
[b:       1 byte ] 13       [b:       1 byte ] 17
[padding: 3 bytes] 16       [padding: 7 bytes] 24
```

如果子类首个成员变量是 long 或者 double 等 8 字节数据类型，而父类结束时没有 8 位对齐。会把子类的小于 8 字节的实例成员先排列，直到能 8 字节对齐。

```java
class A {
    byte a;
}

class B extends A{
    long b;
    short c;  
    byte d;
}
```

内存结构如下:

```text
32 bit                  64bit +UseCompressedOops

[HEADER:  8 bytes]  8       [HEADER:  8 bytes] 12
[a:       1 byte ]  9       [a:       1 byte ] 13
[padding: 3 bytes] 12       [padding: 3 bytes] 16
[c:       2 bytes] 14       [b:       8 bytes] 24
[d:       1 byte ] 15       [c:       4 byte ] 28
[padding: 1 byte ] 16       [d:       1 byte ] 29
[b:       8 bytes] 24       [padding: 3 bytes] 32
```

## Padding(对齐填充)

对齐填充是底层CPU数据总线读取内存数据时的要求，例如，通常CPU按照一行32bit(8byte)读取，如果一个完整的数据体不对齐，那么在内存中存储时，其地址有极大可能横跨两行，而cpu按固定长度行读取，需要读取两行数据，然后合并舍弃掉多余的部分。但如果进行了对齐，则一次性即可取出目标数据，将会大大节省CPU资源。

在hotSpot虚拟机中，默认的对齐位数是8byte，与CPU架构无关。换句话说就是对象的大小必须是8byte的整数倍。因此当对象没有对齐的话，就需要通过对齐填充来补全。

虽然我们在文章开头描述结构的时候将padding放在了最后面。但是实际上**Padding可以在对象头后面，在Instance Data 中间**。

## 指针压缩

指针压缩指对象引用(指针)压缩。对于32bit长度的对象引用，我们可以描述4GB地址空间(2^32=4GB)的任一地址。由于内存地址的基本单元cell是1byte(8bit)，实际上只需要29bit长度就可以描述所有cell地址。也就是说描述一个满4G的地址，最大需要0.5G空间存储对象引用。

如果需要更大的内存地址空间，最简单的方法就是将对象引用长度改为64bit。但是64位JVM在支持更大内存空间的同时，却带来了性能问题：

- 降低CPU缓存命中率: 对象引用增大到64位，CPU能缓存的oop将会更少，从而降低了CPU缓存的效率。
- 增加了GC开销: 64位对象引用需要占用更多的堆空间，留给其他数据的空间将会减少，从而加快了GC的发生，更频繁的进行GC。

因此，JVM提供了指针压缩的功能，使用32位对象引用来描述更大的地址空间。指针压缩时，堆中的引用其实还是按照0x0、0x1、0x2...进行存储地址值。只不过当引用被用于寻址时，JVM将其左移3位。0x0、0x1、0x2...分别被转换为0x0、0x8、0x16。同样在地址被存储为对象指针时，JVM将其右移3位。通过上面的方法，可以通过32位长度的指针描述32*8长度的地址(原地址的递增是1，新地址的递增是8，因为对象地址是8byte对齐)，也就是32G(2^35=32G).

Oracle JDK从1.6.0_23开始在64位系统上会默认开启压缩指针。32位HotSpot VM是不支持`-XX:+UseCompressedOops`参数的，只有64位HotSpot VM才支持。对于大小在4G和32G之间的堆，默认使用开启压缩，当内存超过32G时，压缩参数不生效。

在VM启动的时候，可以设置 `-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompressedOopsMode` 参数来确认压缩指针的工作模式。

### 对齐长度

经过上面的介绍，我们可以发现指针压缩的shift长度和对齐长度N是紧密相关的(2^shift=N)。我们可以通过`-XX:ObjectAlignmentInBytes` 配置对象对齐长度。XX:ObjectAlignmentInBytes可以设置为 8 的整数倍，最大 256。也就是如果配置-XX:ObjectAlignmentInBytes为 16，那么配置最大堆内存超过 64 GB 压缩指针才会失效。

### klass压缩

在jdk15之前 `-XX:+UseCompressedOops`和`-XX:+UseCompressedClassPointers`(压缩类指针)选项的值是相同的。

但是在jdk15后，要让klass压缩失效，需要设置单独设置`-XX：-UseCompressedClassPointers`。禁用UseCompressedOops并不意味着UseCompressedClassPointers也被禁用。



## 数组内存占用

数组也是对象，故有对象的头部，另外数组还有一个记录数组长度的 int 类型。

- 32 位: 8 字节的对象头部，4 字节的 int 长度, 一共是12 字节，8byte对齐后是16字节。
- 64 位+UseCompressedOops: 12 字节的对象头部，4 字节的 int 长度，一共16字节长度。
- 64 位-UseCompressedOops: 16 字节头部，4 字节的 int 长度信息，20 字节，8byte对齐后 24 字节。

随后是每一个数组的元素：基本数据类型或者引用。


## 字符串内存占用

|FieldType|64 bit -UseCompressedOops|64 bit +UseCompressedOops|32 bit|
|:---|:---|:---|:---|
|HEADER|16|12|8|
|value|char[]|8|4|4|
|offset|int|4|4|4|
|count|int|4|4|4|
|hash|int|4|4|4|
|PADDING|4|4|0|
|TOTAL|40|32|24|

不计算 value 引用的 Retained heap size, 字符串本身就需要 24 ~ 40 字节大小。

## 包装类内存占用

包装器的大小 = 对象头对象+内部原语字段大小+内存对齐，下表是具体大小。

|Type|Primitive Size|Header 32|	Header 64|Total size 32|Total size 64|Total size 32 bit with padding|Total size 64 bit with padding|
|:---|:---|:---|:---|:---|:---|:---|:---|
|Byte|1|8|12|9|13|16|16|
|Boolean|1|8|12|9|13|16|16
|Integer|4|8|12|12|16|16|16
|Float|4|8|12|12|16|16|16
|Short|2|8|12|10|14|16|16
|Char|2|8|12|10|14|16|16
|Long|8|8|12|16|20|16|24
|Double|8|8|12|16|20|16|24

可以看到使用包装类存储数据时会带来额外的内存占用。因此在处理大数据量存储时，应该使用基于基类实现的集合，以节省内存开销(降低2/3)。下面是常用的两个框架。

- [eclipse/eclipse-collections](https://github.com/eclipse-collections/eclipse-collections)
- [vigna/fastutil](https://github.com/vigna/fastutil)


## 参考资料

- [1] [OpenJDK.oop](https://hg.openjdk.org/jdk/jdk/file/19afeaa0fdbe/src/hotspot/share/oops/oop.hpp#l52)
- [2] [Java Objects Inside Out](https://shipilev.net/jvm/objects-inside-out/)
- [3] [ObjectHeader32.txt](https://gist.github.com/arturmkrtchyan/43d6135e8a15798cc46c#file-objectheader64-txt-L15)
- [4] [Oracle JDK从6 update 23开始在64位系统上会默认开启压缩指针](https://www.iteye.com/blog/rednaxelafx-1010079)
- [5] [OpenJDK.CompressedOops](https://wiki.openjdk.org/display/HotSpot/CompressedOops)

