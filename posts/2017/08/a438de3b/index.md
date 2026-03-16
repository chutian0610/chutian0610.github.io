# Java虚拟机-虚拟机栈

在前面的[Java虚拟机-内存布局](/posts/2017/08/72bd051d/)一文中我们简单介绍了虚拟机栈：

- java虚拟机栈是线程私有的，他与线程的声明周期同步。虚拟机栈描述的是java方法执行的内存模型。
- 每个方法执行都会创建一个栈帧，栈帧包含局部变量表、操作数栈、动态连接、方法出口等。

本文将继续详细介绍虚拟机栈。

<!--more-->

![](stack.webp)

## 代码和字节码

本文使用代码示例:

```java
package info.victorchu.j8.jvm.stack;
// 用于分析虚拟机栈的代码
public class VMStack {
    
    private static int increaseByTen(int a, int b){
        return a*b + 10; 
    }
    public void test() {
        long c;
        int a, b;
        a = 1;
        b = 2;
        c = increaseByTen(a, b);
        c = c*(a+b);
    }
}
```

通过`javap -verbose`得到代码对应的字节码

```java
public class info.victorchu.j8.jvm.stack.VMStack
  minor version: 0
  major version: 52
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #1                          // info/victorchu/jdk/lab/jvm/stack/VMStack
  super_class: #3                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 3, attributes: 1
Constant pool:
   #1 = Class              #2             // info/victorchu/jdk/lab/jvm/stack/VMStack
   #2 = Utf8               info/victorchu/jdk/lab/jvm/stack/VMStack
   #3 = Class              #4             // java/lang/Object
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Methodref          #3.#9          // java/lang/Object."<init>":()V
   #9 = NameAndType        #5:#6          // "<init>":()V
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Linfo/victorchu/jdk/lab/jvm/stack/VMStack;
  #14 = Utf8               increaseByTen
  #15 = Utf8               (II)I
  #16 = Utf8               a
  #17 = Utf8               I
  #18 = Utf8               b
  #19 = Utf8               test
  #20 = Methodref          #1.#21         // info/victorchu/jdk/lab/jvm/stack/VMStack.increaseByTen:(II)I
  #21 = NameAndType        #14:#15        // increaseByTen:(II)I
  #22 = Utf8               c
  #23 = Utf8               J
  #24 = Utf8               SourceFile
  #25 = Utf8               VMStack.java
{
  public info.victorchu.j8.jvm.stack.VMStack();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #8                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 2: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Linfo/victorchu/jdk/lab/jvm/stack/VMStack;

  private static int increaseByTen(int, int);
    descriptor: (II)I
    flags: (0x000a) ACC_PRIVATE, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=2
         0: iload_0
         1: iload_1
         2: imul
         3: bipush        10
         5: iadd
         6: ireturn
      LineNumberTable:
        line 5: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0     a   I
            0       7     1     b   I

  public void test();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=4, locals=5, args_size=1
         0: iconst_1
         1: istore_3
         2: iconst_2
         3: istore        4
         5: iload_3
         6: iload         4
         8: invokestatic  #20                 // Method increaseByTen:(II)I
        11: i2l
        12: lstore_1
        13: lload_1
        14: iload_3
        15: iload         4
        17: iadd
        18: i2l
        19: lmul
        20: lstore_1
        21: return
      LineNumberTable:
        line 10: 0
        line 11: 2
        line 12: 5
        line 13: 13
        line 14: 21
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0      22     0  this   Linfo/victorchu/jdk/lab/jvm/stack/VMStack;
           13       9     1     c   J
            2      20     3     a   I
            5      17     4     b   I
}
SourceFile: "VMStack.java"
```

### 用到的字节码指令介绍

内存相关:

- iconst_1: 加载int值 1 加载到栈上
- istore_3: 将int值(取栈顶)写入局部变量3
- istore x: 将int值(取栈顶)写入局部变量4
- iload_3:  读取本地变量3，压栈
- iload x:  读取本地变量x，压栈
- invokestatic: 调用静态方法
- bipush x: 将byte值x作为integer值压栈
- ireturn: 返回一个int型数值(从栈顶)                      
- return: 返回void 

其他操作：

- i2l: 将int值转为long
- lmul: long乘long
- iadd: int + int

## 栈帧

每一个方法的执行到执行完成，对应着一个栈帧在虚拟机中从入栈到出栈的过程。

- java虚拟机栈栈顶的栈帧就是当前执行方法的栈帧，PC寄存器会指向该地址。
- 当这个方法调用其他方法的时候就会创建一个新的栈帧，这个新的栈帧会被方法Java虚拟机栈的栈顶，变为当前的活动栈，称为当前栈帧(Current Stack Frame)，这个栈帧所关联的方法称为当前方法(Current Method)。执行引擎所运行的所有字节码指令都只针对当前栈帧进行操作。
- 当这个栈帧所有指令都完成的时候，这个栈帧被移除，之前的栈帧变为活动栈，前面移除栈帧的返回值变为这个栈帧的一个操作数。
- 在编译代码的时候，栈帧中需要多大的局部变量表，多深的操作数栈都已经完全确定了，并且写入到了方法表的Code属性中，因此一个栈帧需要分配多少内存，不会受到程序运行期变量数据的影响，而仅仅取决于具体虚拟机的实现。 

### 局部变量表

局部变量表：Local Variables。定义为一个数字数组，主要用于存储方法参数和定义在方法体内的局部变量这些数据类型包括各类基本数据类型、对象引用（reference），以及returnAddress类型。

- 局部变量表是线程私有的。
- 局部变量表所需的容量大小是在编译期确定下来的，并保存在方法的Code属性的locals数据项中。在方法运行期间是不会改变局部变量表的大小的。
- 局部变量表中的变量只在当前方法调用中有效。在方法执行时，虚拟机通过使用局部变量表完成参数值到参数变量列表的传递过程。当方法调用结束后，随着方法栈帧的销毁，局部变量表也会随之销毁。
- 局部变量存放在局部变量表的slot（变量槽）中，局部变量是编译期可知的各种基本数据类型（8种），引用类型（reference），returnAddress类型的变量。
- 在局部变量表里，32位以内的类型只占用一个slot（包括returnAddress类型）。
- 对于64位长度的数据类型（long，double），虚拟机会以高位对齐方式为其分配两个连续的Slot空间，也就是相当于把一次long和double数据类型读写分割成为两次32位读写。
- JVM会为局部变量表中的每一个Slot都分配一个访问索引，通过这个索引即可成功访问到局部变量表中指定的局部变量值。
- Slot是可以重用的，当Slot中的变量超出了作用域，那么下一次分配Slot的时候，将会覆盖原来的数据。Slot对对象的引用会影响GC（要是被引用，将不会被回收）。  系统不会为局部变量赋予初始值（实例变量和类变量都会被赋予初始值）。也就是说不存在类变量那样的准备阶段。

例如上面示例代码中test方法:

```sh
LocalVariableTable:
  Start  Length  Slot  Name   Signature
      0      22     0  this   Linfo/victorchu/jdk/lab/jvm/stack/VMStack;
     13       9     1     c   J
      2      20     3     a   I
      5      17     4     b   I
```

long的类型签名是J，可以看到局部变量c占据的slot是1和2。使用1索引。Start和Length表示变量在方法类的作用域(指令起点和区间长度)。

### 操作数栈

操作数栈(Operand Stack)也常称为操作栈，它是一个后入先出栈(LIFO)。操作数栈主要用于保存计算过程的中间结果，同时作为计算过程中变量临时的存储空间。同局部变量表一样，操作数栈的最大深度也在编译的时候写入到方法的Code属性的stack数据项中。

- 操作数栈在方法执行过程中，根据字节码指令，往栈中写入数据或提取数据，即入栈(push)和出栈(pop)。操作数栈并非采用访问索引的方式来进行数据访问的，而是只能通过标准的入栈和出栈操作来完成一次数据访问。
- 操作数栈的每一个元素可以是任意Java数据类型，32位的数据类型占一个栈容量，64位的数据类型占2个栈容量,且在方法执行的任意时刻，操作数栈的深度都不会超过stack中设置的最大值。
- 如果被调用的方法带有返回值的话，其返回值将会被压入当前栈帧的操作数栈中，并更新PC寄存器中下一条需要执行的字节码指令。

例如上面示例代码中test方法:

```sh
Code:
  stack=4, locals=5, args_size=1
    0: iconst_1
    1: istore_3
    2: iconst_2
    3: istore        4
    5: iload_3
    6: iload         4
    8: invokestatic  #20 // Method increaseByTen(II)I
    11: i2l
    12: lstore_1
    13: lload_1
    14: iload_3
    15: iload        4
    17: iadd
    18: i2l
    19: lmul
    20: lstore_1
    21: return
LocalVariableTable:
  Start  Length  Slot  Name   Signature
      0      22     0  this   Linfo/victorchu/jdk/lab/jvm/stack/VMStack;
     13       9     1     c   J
      2      20     3     a   I
      5      17     4     b   I
```

操作数栈变化

|id|指令|操作数栈|局部变量表|
|:---|:---|:---|:---|
|0|iconst_1|[1]|{this}|
|1|istore_3|[]|{this,a=1}|
|2|iconst_2|[2]|{this,a=1}|
|3|istore 4|[]|{this,a=1,b=2}|
|5|istore 4|[1]|{this,a=1,b=2}|
|6|istore 4|[1,2]|{this,a=1,b=2}|
|8|invokestatic|[12]|{this,a=1,b=2}|
|11|i2l|[12L]|{this,a=1,b=2}|
|12|lstore_1|[]|{this,a=1,b=2,c=12L}|
|13|lload_1|[12L]|{this,a=1,b=2,c=12L}|
|14|iload_3|[1,12L]|{this,a=1,b=2,c=12L}|
|15|iload 4|[2,1,12L]|{this,a=1,b=2,c=12L}|
|17|iadd|[3,12L]|{this,a=1,b=2,c=12L}|
|18|i2l|[3L,12L]|{this,a=1,b=2,c=12L}|
|19|lmul|[36L]|{this,a=1,b=2,c=12L}|
|20|lmul|[]|{this,a=1,b=2,c=36L}|
|21|return|[]|{this,a=1,b=2,c=36L}|

### 动态链接

动态链接（Dynamic Linking）: 每个栈帧都保存了 一个 可以指向当前方法所在类的 运行时常量池, 目的是: 当前方法中如果需要调用其他方法的时候, 能够从运行时常量池中找到对应的符号引用, 然后将符号引用转换为直接引用,然后就能直接调用对应方法, 这就是动态链接。例如test方法中的invokestatic。

```sh
invokestatic  #20 // Method increaseByTen(II)I
```

后面的#20就是常量池的索引:

```java
Constant pool:
   #1 = Class              #2             // info/victorchu/jdk/lab/jvm/stack/VMStack
   #2 = Utf8               info/victorchu/jdk/lab/jvm/stack/VMStack
   #3 = Class              #4             // java/lang/Object
   #4 = Utf8               java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Methodref          #3.#9          // java/lang/Object."<init>":()V
   #9 = NameAndType        #5:#6          // "<init>":()V
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               Linfo/victorchu/jdk/lab/jvm/stack/VMStack;
  #14 = Utf8               increaseByTen
  #15 = Utf8               (II)I
  #16 = Utf8               a
  #17 = Utf8               I
  #18 = Utf8               b
  #19 = Utf8               test
  #20 = Methodref          #1.#21         // info/victorchu/jdk/lab/jvm/stack/VMStack.increaseByTen:(II)I 
  //-- 这里存储了increaseByTen的方法符号引用
  #21 = NameAndType        #14:#15        // increaseByTen:(II)I 
  //-- 方法签名
  #22 = Utf8               c
  #23 = Utf8               J
  #24 = Utf8               SourceFile
  #25 = Utf8               VMStack.java
```

- 静态链接： 当一个字节码文件被装载进JVM内部时，如果被调用的目标方法在编译期克制，且运行期保持不变时，这种情况下降调用方法的符号引用转换为直接引用的过程称之为静态链接
- 动态链接：如果被调用的方法在编译期无法被确定下来，也就是说，只能够在程序运行期将调用的方法的符号转换为直接引用，由于这种引用转换过程具备动态性，因此也被称之为动态链接。

### 方法返回地址

存放调用该方法的pc寄存器的值。当一个方法开始执行后，只有两种方式可以退出这个方法：

- 正常完成出口：执行引擎遇到任意一个方法返回的字节码指令（return），会有返回值传递给上层的方法调用者，简称正常完成出口；究竟需要使用哪一个返回指令，还需要根据方法返回值的实际数据类型而定。
- 异常完成出口 ：在方法执行过程中遇到异常（Exception），并且这个异常没有在方法内进行处理，也就是只要在本方法的异常表中没有搜索到匹配的异常处理器，就会导致方法退出，简称异常完成出口。

无论通过哪种方式退出，在方法退出后都返回到该方法被调用的位置。方法正常退出时，调用者的pc计数器的值作为返回地址，即调用该方法的指令的下一条指令的地址。而通过异常退出的，返回地址是要通过异常表来确定，栈帧中一般不会保存这部分信息。

## Java中如何查看栈帧

java中可以通过下面的代码获取当前代码栈帧:

```java
StackTraceElement[] stackTraces = Thread.currentThread().getStackTrace();
```

每个 StackTraceElement 对象代表一个独立的栈帧，所有栈帧的顶部是getStackTrace方法。

StackTraceElement记录了:

- 类加载器的名称
- 模块名称
- 模块版本
- 声明方法的类
- 方法名称
- 文件名称
- 行号

下面看一个使用例子:

```java
public class VMStackPrint
{
    public static void main(String[] args)
    {
        factorial(3);
    }

    static void printStackFrame(){
        StackTraceElement[] stackTraces = Thread.currentThread().getStackTrace();
        // 此处从2开始，是因为0是getStackTrace()
        // 而1是printStackFrame();
        for (int i = 2; i < stackTraces.length; i++) {
            String prefix = i<=2?"":String.join("",Collections.nCopies(i-3,"  "))+"└─";
            System.out.println(prefix+stackTraces[i].toString());
        }
    }

    static int factorial(int n)
    {
        printStackFrame();
        if(n<= 0){
            return 1;
        }
        return n * factorial(n-1);
    }
}
```

可以得到栈帧结果:

```
/* =================== frame1 =======================*/
info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:25)
└─info.victorchu.j8.jvm.stack.VMStackPrint.main(VMStackPrint.java:12)
/* =================== frame2 =======================*/
info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:25)
└─info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:29)
  └─info.victorchu.j8.jvm.stack.VMStackPrint.main(VMStackPrint.java:12)
/* =================== frame3 =======================*/  
info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:25)
└─info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:29)
  └─info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:29)
    └─info.victorchu.j8.jvm.stack.VMStackPrint.main(VMStackPrint.java:12)
/* =================== frame4 =======================*/     
info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:25)
└─info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:29)
  └─info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:29)
    └─info.victorchu.j8.jvm.stack.VMStackPrint.factorial(VMStackPrint.java:29)
      └─info.victorchu.j8.jvm.stack.VMStackPrint.main(VMStackPrint.java:12)
```

## 参考

- [1] [Oracle Jvm - Frames](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.6)

- [2] [Java Bytecode instructions - Wikipedia](https://en.wikipedia.org/wiki/List_of_Java_bytecode_instructions)

