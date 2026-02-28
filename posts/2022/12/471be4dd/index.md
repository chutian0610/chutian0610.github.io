# JAVA and SIMD

本文简单介绍了SIMD技术和SIMD在Java中的应用。SIMD(Single Instruction Multiple Data)即单指令流多数据流，是一种采用一个控制器来控制多个处理器，同时对一组数据（又称“数据向量”）中的每一个分别执行相同的操作从而实现空间上的并行性的技术。简单来说就是一个指令能够同时处理多个数据。

<!--more-->

## 什么是SIMD

通常程序代码是被串行执行的。计算型程序通常需要在数据上做大量计算。一般情况下多数数据需要被相同的方式处理。例如，对于1000个粒子的模拟，模拟中会有一个步骤，用其当前速度更新每个粒子的位置 `v: s[i] = s[i] + v[i]`. 从`s[0]`到`s[999]`每个粒子都要进行这个操作。  

如果可以将多个粒子的计算结合起来一次计算，这样的话就可以实现加快计算的目的。例如可以将4个粒子的计算放在一个组中如下：

```
s[0] = s[0] + v[0];
s[1] = s[1] + v[1];
s[2] = s[2] + v[2];
s[3] = s[3] + v[3];
```

也就是说在多个数据上一次性执行一个操作。 在汇编代码级别，有专门针对这些分组操作的说明。此概念被叫做single instruction, multiple data简称SIMD。

![](figure6.gif)

从上图中可以看出，SIMD机器是并行方式利用数据，这一属性也被称为数据并行性，当有大量统一类型的数据需要对其执行相同的指令时，可以利用数据并行性特点来完成。与之相似的一个概念是指令级并行性(LIP),这是超标量SISD机器的一个特性。简单来说就是在同一数据流上一次性执行多条指令。SIMD 是关于利用数据流中的并行性，而超标量 SISD 是关于利用指令流中的并行性。

### 矢量化

SIMD是从指令设计者(即CPU制造商)的角度给出的概念名称。但这不是唯一的视角，在数学上，由固定数量的元素组成的有序组(`s[0..3]` 和 `v[0..3]`) 被称作向量。所以SIMD也被叫做向量指令。这只是对同一件事的另一个观点，是从使用指令的用户观点来看的。矢量化是使用矢量指令来加速程序执行，将标量操作转化为矢量操作。矢量化可以由程序员完成，也可以由编译器自动实现。在后一种情况下，它被称为自动矢量化(auto vectorization)。

>  Auto vectorization is a kind of code optimization which is done by a compiler, either by an AOT compiler at compile time, or by a JIT compiler at execution time.

## Java中的向量

编写Java程序后，Java文件中的Java源代码被编译成字节码并保存到类文件中。然后，在程序执行之前或执行过程中，它的字节码通常会再次编译，这次是从字节码到本机机器码。后一种编译通常在程序执行时进行，因此它是JIT编译（ JIT compilation）。   

在Java中，目前矢量化不是由程序员完成的，而是由编译器自动完成的。编译器接受标准Java字节码，并自动确定哪些部分可以转换为向量指令。像OpenJDK或Oracle的Java这样的通用Java环境可以生成矢量化的机器代码。

### 可能利用矢量化的代码

如果代码对数组的许多连续元素执行相同的操作，则可以将其转换为矢量化指令。例如:

```java
for (int i = 0; i < x.length; i++) {
    z[i] = x[i] * y[i];
}
```

Java可能会将此类循环矢量化，将其转化为如下伪码。

```java
for (int i = 0; i < x.length; i += 4) {
    Load x[i, i+1, i+2, i+3] into xmm0
    Load y[i, i+1, i+2, i+3] into xmm1
    Multiply xmm0 * xmm1 and store result in xmm0
    Store xmm0 into z[i, i+1, i+2, i+3]
}
// Multiply 向量化乘法
```

> 此优化可以通过JVM(Hotspot)开关 `-XX：+UseSuperWord`打开/关闭，该开关默认打开。


### 内置函数

在 jdk9 之后，jdk 很多的[内置函数](https://github.com/openjdk/jdk/blob/master/src/hotspot/share/classfile/vmIntrinsics.hpp)都有`@HotSpotIntrinsicCandidate`这个注解，他的含义是在 JVM 会尝试将此函数实现替换为基于 CPU 指令的更加高效的实现方式。例如我们看StringLatin1.indexOf函数，会调用到这个函数

```java
@HotSpotIntrinsicCandidate
public static int indexOf(byte[] value, int valueCount, byte[] str, int strCount, int fromIndex)
```
内容看起来只是普通的字符串匹配的 java 代码，但实际运行时会替换为基于 SSE4.2 指令集中的 PCMPESTRI 指令，在 16 字节内的字符中查找另一个16字节以内的字符，并返回命中时的索引。相比之前的单个字符配置，理论性能差距有16倍。

## 检查向量化是否被使用

### 准备一个Mirco基准

为了生成汇编级别的向量指令，我们首先需要创建一个可以从向量指令中获益且可以运行的java程序。 如下：

```java
/**
 * Run with this command to show native assembly:<br/>
 * Java -XX:+UnlockDiagnosticVMOptions
 * -XX:CompileCommand=print,VectorizationMicroBenchmark.square
 * VectorizationMicroBenchmark
 */
public class VectorizationMicroBenchmark {
 
    private static void square(float[] a) {
        for (int i = 0; i < a.length; i++) {
            a[i] = a[i] * a[i]; // line 11
        }
    }
 
    public static void main(String[] args) throws Exception {
        float[] a = new float[1024];
 
        // repeatedly invoke the method under test. this
        // causes the JIT compiler to optimize the method
        for (int i = 0; i < 1000 * 1000; i++) {
            square(a);
        }
    }
}
```
上面的程序可以使编译器把square进行优化。

### 运行Mirco基准

1. 打开 IDE 并创建一个新项目。在新项目中创建一个新类并将其命名为VectorizationMicroBenchmark,将代码片段 2 的代码复制粘贴到其中。
2. 在 IDE 中修改运行配置- "VM 参数"下：输入以下内容:`-XX:+UnlockDiagnosticVMOptions -XX:CompileCommand="print info.victorchu.jdk.lab.simd.VectorizationMicroBenchmark::square"`

3. 获取libhsdis并将文件`hsdis-amd64.so`复制到Java-home/lib/*/server目录中.
4. 再次运行 VectorizationMicroBenchmark

步骤 4.将大量信息打印到控制台，其中一部分是反汇编的本机机器代码。如果您看到很多消息但没有汇编指令，如mov, push, add等，那么也许您可以在输出中的某处找到以下消息：`Could not load hsdis-amd64.so; library not loadable; PrintAssembly is disabled`。

如果您看到此消息，这意味着Java找不到文件hsdis-amd64.so-它不在正确的目录中或没有正确的名称。在Linux上，创建符号链接时也会发生这种情况。Java在这里不接受符号链接。您必须复制该文件。hsdis-amd64.so是显示生成的本机代码所需的反汇编程序。JIT编译器将Java字节码编译为本机机器码后，hsdis-amd64.so用于反汇编本机机器码，使其具有可读性。

### 输出说明

在输出中找到汇编指令后，可能会惊讶地发现，您不仅找到了square方法的汇编代码，而且你发现了它的多个版本。这是因为JIT编译器在第一次运行时没有完全优化该方法。在调用该方法之后，它将其编译为本机代码而不进行优化。在多次调用之后，它通过一些优化（但不是全部优化）再次编译该方法。只有在几千次调用之后，编译器才确信该方法非常重要，需要在编译时启用所有优化，包括向量化。因此，最好的编译通常是输出中的最后一个编译。

开始搜索输出末尾的"第11行"，向后搜索。您可能会发现如下内容：

```

0x...ac70: vmovss  0x10(%rbx,%rbp,4),%xmm0  ;*faload {reexecute=0 rethrow=0 return_oop=0}
                                     ; - VectorizationMicroBenchmark::square@9 (line 11)
 
0x...ac76: vmulss  %xmm0,%xmm0,%xmm1
0x...ac7a: vmovss  %xmm1,0x10(%rbx,%rbp,4)  ;*fastore {reexecute=0 rethrow=0 return_oop=0}
                                     ; - VectorizationMicroBenchmark::square@14 (line 11)
```

请注意代码段3中末尾带有-ss的指令vmulss。

```
vmulss: multiply scalar single-precision floating-point values
```

vmulss仅将一个浮点数与另一个浮点数相乘。所以这不是我们想要的。（这里，标量表示只有一个，单精度表示32位，即浮点而不是双精度）。相反，我们希望找到一条指令，它一次将许多浮点与许多其他浮点相乘。所以请继续关注。你最终会发现：

```
0x...ac54: vmovdqu 0x10(%rbx,%rbp,4),%ymm0  ;*faload {reexecute=0 rethrow=0 return_oop=0}
                                     ; - VectorizationMicroBenchmark::square@9 (line 11)

0x...ac5a: vmulps  %ymm0,%ymm0,%ymm0
0x...ac5e: vmovdqu %ymm0,0x10(%rbx,%rbp,4)  ;*fastore {reexecute=0 rethrow=0 return_oop=0}
                                     ; - VectorizationMicroBenchmark::square@14 (line 11)
```


`vmulps: multiply packed single-precision floating-point values`. vmulps是一条真正的SIMD指令。这里，(packed)打包是指在一个寄存器中打包在一起的多个元素。这表明应用了自动矢量化。


## 参考

- [1] [Basics of SIMD Programming](http://ftp.cvut.cz/kernel/people/geoff/cell/ps3-linux-docs/CellProgrammingTutorial/BasicsOfSIMDProgramming.html)
- [2] [SIMD 架构](https://arstechnica.com/features/2000/03/simd/)
- [3] [JAVA and SIMD](https://github.com/Teradata/presto-tech/blob/master/blogs/simd.md)

