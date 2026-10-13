# 字节序

字节(英语：Byte)，通常用作计算机信息计量单位，不分数据类型，是通信和数据存储的概念。一个字节代表八个比特。比特（英语：Bit，亦称二进制位）指二进制中的一位，是信息的最小单位。那么多个字节如何表示，接下来本文将介绍字节顺序。

<!--more-->

## 字节顺序

字节顺序，又称端序或尾序（英语：Endianness），在计算机科学领域中，指电脑内存中或在数字通信链路中，组成多字节的字的字节的排列顺序。

在几乎所有的机器上，多字节对象都被存储为连续的字节序列。例如在C语言中，一个类型为int的变量x地址为0x100，那么其对应地址表达式&x的值为0x100。且x的四个字节将被存储在电脑内存的0x100, 0x101, 0x102, 0x103位置。

字节的排列方式有两个通用规则。例如，将一个多位数的低位放在较小的地址处，高位放在较大的地址处，则称小端序；反之则称大端序。在网络应用中，字节序是一个必须被考虑的因素，因为不同机器类型可能采用不同标准的字节序，所以均按照网络标准转化。

例如假设上述变量x类型为int，位于地址0x100处，它的值为0x01234567，地址范围为0x100~0x103字节，其内部排列顺序依赖于机器的类型。大端法从首位开始将是：`0x100: 0x01`,`0x101: 0x23`,..。而小端法将是：`0x100: 0x67`, `0x101: 0x45`,..。

对于单一的字节（a byte），大部分处理器以相同的顺序处理位元（bit），因此单字节的存放方法和传输方式一般相同。

对于多字节数据，如整数（32位机中一般占4字节），在不同的处理器的存放方式主要有两种，以内存中0x0A0B0C0D的存放方式为例[^1]，分别有以下几种方式：

### 大端序

数据以8bit为单位，最高位字节是0x0A 存储在最低的内存地址处。下一个字节0x0B存在后面的地址处。

![](Big-Endian.svg)

网络传输一般采用大端序，也被称之为网络字节序，或网络序。IP协议中定义大端序为网络字节序。

### 小端序

数据以8bit为单位,最低位字节是0x0D 存储在最低的内存地址处。后面字节依次存在后面的地址处。

![](Little-Endian.svg)

### 混合序

混合序（英：middle-endian）具有更复杂的顺序。以PDP-11为例，`0x0A0B0C0D`被存储为:高16bit和低16bit以大端序存储，但16bit内部以小端存储。

```sh
# 地址增长方向  →  
|0x0B|0x0A|0x0D|0x0C|
```

## JVM

- class 文件中的多字节数据以大端方式存储

A class file consists of a stream of 8-bit bytes. All 16-bit, 32-bit, and 64-bit quantities are constructed by reading in two, four, and eight consecutive 8-bit bytes, respectively. Multibyte data items are always stored in big-endian order, where the high bytes come first. In the Java SE platform, this format is supported by interfaces java.io.DataInput and java.io.DataOutput and classes such as java.io.DataInputStream and java.io.DataOutputStream[^2].

- 字节代码指令中的操作也是大端的,如果它跨越多个字节

If an operand is more than one byte in size, then it is stored in big-endian order-high-order byte first. For example, an unsigned 16-bit index into the local variables is stored as two unsigned bytes, byte1 and byte2, such that its value is `(byte1 << 8) | byte2`[^3].

- 可以通过 java.nio.ByteOrder.nativeOrder() 获取机器的大小端定义。


[^1]: 0x前缀代表十六进制
[^2]: The Java Virtual Machine Specification, Java SE 7 Edition, Chapter 4: The class File Format。
[^3]: The Java Virtual Machine Specification, Java SE 7 Edition, Section 2.11: Instruction Set Summary。

