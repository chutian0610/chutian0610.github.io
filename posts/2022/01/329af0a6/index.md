# Flink:Buffer框架

Flink是使用 JVM 的大数据开源计算框架，基于 JVM 的数据分析引擎都需要面对将大量数据存到内存中，这就不得不面对 JVM 存在的几个问题：

1. Java 对象存储密度低。一个只包含 boolean 属性的对象占用了16个字节内存: 对象头占了8个,boolean 属性占了1个,对齐填充占了7个。而实际上只需要一个bit就够了。
- Full GC 会极大地影响性能, 尤其是为了处理更大数据而开了很大内存空间的JVM来说, GC 会达到秒级甚至分钟级。
- OOM 问题影响稳定性。OutOfMemoryError是分布式计算框架经常会遇到的问题, 当JVM中所有对象大小超过分配给JVM的内存大小时, 就会发生OutOfMemoryError错误, 导致JVM崩溃, 分布式框架的健壮性和性能都会受到影响。

对于第一个问题，如果采用基类存储就可以解决。而第二个问题，可以考虑是使用直接内存和内存池来解决 Full GC 的问题。OOM 问题需要支持内存数据溢写到磁盘，即支持内存数据的序列化和反序列化。这里不使用 JDK 原始 buffer 的原因是 JDK Buffer只支持存储相同固定类型的实例数据，而实际上流式数据处理的总是一行数据，且数据要支持可扩展的类系统。

因此，Flink 选择了实现自己管理的内存单元和可扩展的类型系统，也就是接下来介绍的 Buffer框架(Memory Segment) 和对应的 TypeSerializer。

![](memory-mgmt.webp)

<!--more-->

## MemorySegment

MemorySegment已作为单独的一个类用于处理：堆内内存、堆外直接内存(基于ByteBuffer)或堆外不安全内存(基于 Unsafe 分配的直接内存)。MemorySegment只能通过MemorySegmentFactory创建:

- 基于堆内存
  - wrap: 直接将byte[] buffer数组封装成MemorySegment
  - allocateUnpooledSegment: 分配堆内存空间创建MemorySegment
- 基于直接内存
  - allocateUnpooledOffHeapMemory: 基于 DirectByteBuffer封装成MemorySegment
  - allocateOffHeapUnsafeMemory:   Unsafe分配直接内存创建MemorySegment

### 读写原理

上面介绍了 MemorySegment 同时支持对堆和直接内存的操作。下面就介绍下操作的实现原理:

有几个关键属性:

- heapMemory: 在堆内存模式下，该属性指向堆中的数组。直接内存模式下，为 null。
- address: 基地址
  - 堆内存模式下是 byte[]的arrayBaseOffset
  - 直接内存模式下是直接内存的基地址(如果是基于 ByteBuffer 的直接内存，通过反射获取 ByteBuffer 的 address 属性)

对内存的操作都是通过 UnSafe 的方法实现的，这里以 getInt 为例：

```java
public int getInt(int index) {
    // 计算出数据地址
    final long pos = address + index;
    if (index >= 0 && pos <= addressLimit - 4) {
        // Unsafe 方法读取数据
        // 如果 heapMemory 是 Java 对象，就给定偏移 Pos 处读取属性
        // 如果 heapMemory 为 null，就从直接内存地址 Pos 处读取数据
        return UNSAFE.getInt(heapMemory, pos);
    } else if (address > addressLimit) {
        throw new IllegalStateException("segment has been freed");
    } else {
        // index is in fact invalid
        throw new IndexOutOfBoundsException();
    }
}
```

### JIT 优化

查看源码可以发现 MemorySegment 只有一个单独的 final class，并没有子类实现(例如像 Buffer 一样分为 Heap 实现和 Direct 实现)。这是因为JIT 编译时，所有要调用的方法都是确定的，所有的方法调用都可以被去虚化（de-virtualized）和内联（inlined），这可以极大地提高性能（MemroySegment的使用相当频繁）。然而如果同时加载两个子类，那么 JIT 编译器就只能在真正运行到的时候才知道是哪个子类，这样就无法提前做优化。实际测试的性能差距在 2.7 倍左右。

## 类型系统

目前 Java 生态圈提供了众多的序列化框架：Java serialization, Kryo, Apache Avro 等等。但是 Flink 实现了自己的序列化框架。因为在 Flink 中处理的数据流通常是同一类型，由于数据集对象的类型固定，对于数据集可以只保存一份对象Schema信息，节省大量的存储空间。同时，对于固定大小的类型，也可通过固定的偏移位置存取。当我们需要访问某个对象成员变量的时候，通过定制的序列化工具，并不需要反序列化整个Java对象，而是可以直接通过偏移量，只是反序列化特定的对象成员变量。如果对象的成员变量较多时，能够大大减少Java对象的创建开销，以及内存数据的拷贝大小。

类型信息由 TypeInformation 类表示，TypeInformation 支持以下几种类型：

- BasicTypeInfo: 任意Java 基本类型（装箱的）或 String 类型。
- BasicArrayTypeInfo: 任意Java基本类型数组（装箱的）或 String 数组。
- WritableTypeInfo: 任意 Hadoop Writable 接口的实现类。
- TupleTypeInfo: 任意的 Flink Tuple 类型(支持Tuple1 to Tuple25)。Flink tuples 是固定长度固定类型的Java Tuple实现。
- CaseClassTypeInfo: 任意的 Scala CaseClass(包括 Scala tuples)。
- PojoTypeInfo: 任意的 POJO (Java or Scala)，例如，Java对象的所有成员变量，要么是 public 修饰符定义，要么有 getter/setter 方法。
- GenericTypeInfo: 任意无法匹配之前几种类型的类。

![](type.webp)

前六种数据类型基本上可以满足绝大部分的Flink程序，针对前六种类型数据集，Flink皆可以自动生成对应的TypeSerializer，能非常高效地对数据集进行序列化和反序列化。对于最后一种数据类型GenericTypeInfo，Flink会使用Kryo进行序列化和反序列化。每个TypeInformation中，都包含了serializer，类型会自动通过serializer进行序列化，然后用Java Unsafe接口写入MemorySegments。

下图展示 一个内嵌型的`Tuple3<Integer,Double,Person>`对象的序列化过程。

![](data-serialization.webp)

可以看出这种序列化方式存储密度是相当紧凑的。其中 int 占4字节，double 占8字节，POJO多个一个字节的header，PojoSerializer只负责将header序列化进去，并委托每个字段对应的serializer对字段进行序列化。

Flink 的类型系统可以很轻松地扩展出自定义的TypeInformation、Serializer以及Comparator，来提升数据类型在序列化和比较时的性能。

### 序列化

Flink 中每一个具体的数据类型都对应一个 TypeInformation 的具体实现，每一个 TypeInformation 都会为对应的具体数据类型提供一个专属的序列化器。TypSerializer是所有序列化的基础类，需要实现序列化和逆序列化方法，将数据与DataView进行转换，dataView作为MemorySegment 的抽象显示，序列化与逆序列化就是讲自定义的内存结构与实体进行转换的过程。

```java
public abstract class TypeSerializer<T> implements Serializable {
    public abstract void serialize(T record, DataOutputView target) throws IOException;
    public abstract T deserialize(DataInputView source) throws IOException;
}
```

#### NullSerializer

```java
@Override
public void serialize(Object record, DataOutputView target) throws IOException {
    target.writeByte(0);
}

@Override
public Object deserialize(DataInputView source) throws IOException {
    source.readByte();
    return null;
}
```

#### LongSerializer

```java
@Override
public void serialize(Long record, DataOutputView target) throws IOException {
  target.writeLong(record);
}

@Override
public Long deserialize(DataInputView source) throws IOException {
  return source.readLong();
}
// org.apache.flink.core.memory.DataOutputSerializer implements DataOutputView
public void writeLong(long v) throws IOException {
    if (this.position >= this.buffer.length - 7) {
        resize(8);
    }
    if (LITTLE_ENDIAN) {
        v = Long.reverseBytes(v);
    }
    UNSAFE.putLong(this.buffer, BASE_OFFSET + this.position, v);
    this.position += 8;
}
```

#### StringSerializer

```java
@Override
public void serialize(String record, DataOutputView target) throws IOException {
  StringValue.writeString(record, target);
}

@Override
public String deserialize(DataInputView source) throws IOException {
  return StringValue.readString(source);
}
```
由于 String 是变长的，所以序列化方法比较复杂。

- 先写入长度+1(长度为 0 用来表示 Null 值)作为长度，然后再写数据。
- 以一个byte(8位)为一格，小于128的都好办，大于128的需要循环写入。
- 大于128的字符读取结束的条件是遇到小于128的byte，所以变长字符的写入要大于128，通过每次写入7位，第八位强制设为1实现。

> string值部分的序列化其实是Varint编码方案

StringValue 中定义了一个属性：

```java
private static final int HIGH_BIT = 0x1 << 7;
```

写入的时候，是通过DataOutput.write(byte)写入数据，一次最多只能写入8位数据。flink的做法是每次获取7位的数据，然后第八位通过（ 位或`|` `HIGH_BIT[1000 0000]`） 强制置为1，在二进制中，置为第八位代表正负，1位负，0位正。然后右移7位，获取下一个高7位的数据，以此类推，写入到流中。

```java
public static final void writeString(CharSequence cs, DataOutput out) throws IOException {
  if (cs != null) {
      int strlen = cs.length();

      // the length we write is offset by one, because a length of zero indicates a null value
      int lenToWrite = strlen + 1;
      if (lenToWrite < 0) {
          throw new IllegalArgumentException("CharSequence is too long.");
      }

      // string is prefixed by it's variable length encoded size, which can take 1-5 bytes.
      if (lenToWrite < HIGH_BIT) {
          out.write((byte) lenToWrite);
      } else if (lenToWrite < HIGH_BIT14) {
          out.write((lenToWrite | HIGH_BIT));
          out.write((lenToWrite >>> 7));
      } else if (lenToWrite < HIGH_BIT21) {
          out.write(lenToWrite | HIGH_BIT);
          out.write((lenToWrite >>> 7) | HIGH_BIT);
          out.write((lenToWrite >>> 14));
      } else if (lenToWrite < HIGH_BIT28) {
          out.write(lenToWrite | HIGH_BIT);
          out.write((lenToWrite >>> 7) | HIGH_BIT);
          out.write((lenToWrite >>> 14) | HIGH_BIT);
          out.write((lenToWrite >>> 21));
      } else {
          out.write(lenToWrite | HIGH_BIT);
          out.write((lenToWrite >>> 7) | HIGH_BIT);
          out.write((lenToWrite >>> 14) | HIGH_BIT);
          out.write((lenToWrite >>> 21) | HIGH_BIT);
          out.write((lenToWrite >>> 28));
      }

      // write the char data, variable length encoded
      for (int i = 0; i < strlen; i++) {
          int c = cs.charAt(i);

          // manual loop unroll, as it performs much better on jdk8
          if (c < HIGH_BIT) {
              out.write(c);
          } else if (c < HIGH_BIT14) {
              out.write(c | HIGH_BIT);
              out.write((c >>> 7));
          } else {
              out.write(c | HIGH_BIT);
              out.write((c >>> 7) | HIGH_BIT);
              out.write((c >>> 14));
          }
      }
  } else {
      out.write(0);
  }
}
```

假设写入一个汉字 ：测。这个字对应的`(int)"测".charAt(0)`为27979(110110101001011)，

- 写入的时候，先获取长度+1。第一位为2
- 将27979与HIGH_BIT进行或操作将第八位置为1，得到28107(110110111001011)，其实真实写入的是28107的后八位11001011，右移7位后进行计算得到218(11011010)，再一次得到1，序列化后的是 `2 203 218 1`. 写入byte中得到`2 -53 -38 1`.

下图是不包含长度位的数据部分。

```mermaid
packet-beta
  0-7: "11001011" 
  8-15: "11011010" 
  16: "1"
```

> 因为有符号范围在 -128 ~ 127之间，无符号是在0-256，由于写是默认按照有符号写，所以写11001011会变成-53

逆序列化的时候默认如果小于128，就会当做是结束条件，直接转换成char，所以在变长的string中，还没结束的部分，一定要大于128，这就是为什么写入的时候，一定要强制将第八位置为1.

```java
public static String readString(DataInput in) throws IOException {
  // the length we read is offset by one, because a length of zero indicates a null value
  int len = in.readUnsignedByte();

  if (len == 0) {
      return null;
  }

  if (len >= HIGH_BIT) {
      int shift = 7;
      int curr;
      len = len & 0x7f;
      while ((curr = in.readUnsignedByte()) >= HIGH_BIT) {
          len |= (curr & 0x7f) << shift;
          shift += 7;
      }
      len |= curr << shift;
  }

  // subtract one for the null length
  len -= 1;

  final char[] data;
  if (len > SHORT_STRING_MAX_LENGTH) {
      data = new char[len];
  } else {
      data = charBuffer.get();
  }

  for (int i = 0; i < len; i++) {
      int c = in.readUnsignedByte();
      if (c >= HIGH_BIT) {
          int shift = 7;
          int curr;
          c = c & 0x7f;
          while ((curr = in.readUnsignedByte()) >= HIGH_BIT) {
              c |= (curr & 0x7f) << shift;
              shift += 7;
          }
          c |= curr << shift;
      }
      data[i] = (char) c;
  }

  return new String(data, 0, len);
}
```


- 读取的2-1=1获取数据长度为1
- -53 通过in.readUnsignedByte()得到203,对应11001011，取七位1001011，作为低七位，
继续获取218(11011010)，取七位(1011010)，再进行左移7位操作作为高一阶的七位，拼上低七位，得到10110101001011,最后一个1，进行左移14位操作作为再高一阶的七位，拼上低十四位得到110110101001011对应的十进制为27979，转为测这个char。

#### RowData

flink的 rowData 表示一行数据。[BinaryRowData](https://github.com/apache/flink/blob/master/flink-table/flink-table-common/src/main/java/org/apache/flink/table/data/binary/BinaryRowData.java)是基于 MemorySegment 的实现。

rowData 对于基础类型的 field 会在 MemorySegment 的头部存储NullBitMap.用于辅助判断当前字段是否为 null。

> [BinaryRowWriter](https://github.com/apache/flink/blob/master/flink-table/flink-table-runtime/src/main/java/org/apache/flink/table/data/writer/BinaryRowWriter.java) 中描述了如何构造BinaryRowData

## 参考

- [1] [Juggling with Bits and Bytes](https://flink.apache.org/2015/05/11/juggling-with-bits-and-bytes/)

- [2] [Off-heap Memory in Apache Flink and the curious JIT compiler](https://flink.apache.org/2015/09/16/off-heap-memory-in-apache-flink-and-the-curious-jit-compiler/)

- [3] [Peeking into Apache Flink's Engine Room](https://flink.apache.org/2015/03/13/peeking-into-apache-flinks-engine-room/)

