# 使用 JOL 分析 Java 对象内存

前面介绍了[Java对象在JVM中的内存布局](/posts/2017/04/7937355a/)。那么有什么方法可以方便的计算Java对象的内存占用？

本篇文章我们将介绍如何使用[JOL](https://github.com/openjdk/jol)分析Java对象内存。JOL是一个用来分析JVM中Object布局的小工具。包括Object在内存中的占用情况，实例对象的引用情况等等。JOL可以在代码中使用，也可以独立的以命令行中运行，文中使用java为jdk8。

<!--more-->

## quickstart

项目地址[chutian0610/code-lab](https://github.com/chutian0610/code-lab/tree/main/demos/jol-quickstart)

首先在项目依赖中添加JOL。

```xml
<dependency>
    <groupId>org.openjdk.jol</groupId>
    <artifactId>jol-core</artifactId>
</dependency>
```

然后可以使用ClassLayout分析对象布局。

```java
public class JolSimple
{
    public static void main(String[] args) {
        System.out.println(VM.current().details());
        System.out.println(ClassLayout.parseClass(Object.class).toPrintable());
    }
}
```

默认jol会使用jvm 的attach api动态加载agent。但是常有失败的可能。比较推荐的方法是:

1. 显式指定Manifest

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <executions>
        <execution>
            <id>jol-samples</id>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <transformers>
                    <transformer
                            implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <manifestEntries>
                            <Premain-Class>org.openjdk.jol.vm.InstrumentationSupport</Premain-Class>
                            <Launcher-Agent-Class>org.openjdk.jol.vm.InstrumentationSupport$Installer</Launcher-Agent-Class>
                        </manifestEntries>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```
2. `-javaagent` 方式指定 agent。

## Case

下面将通过一些案例介绍如何使用JOL分析对象内存。下面的case默认开启了内存压缩。

### 基础类型&引用

使用下面的一行代码可以知道基础类型和引用占用的内存大小。

```java
System.out.println(VM.current().details());
```

打印日志如下:

```sh
## VM mode: 64 bits
## Compressed references (oops): 3-bit shift
## Compressed class pointers: 0-bit shift and 0x800000000 base
## Object alignment: 8 bytes
## ref, bool, byte, char, shrt,  int,  flt,  lng,  dbl
## Field sizes:            4,    1,    1,    2,    2,    4,    4,    8,    8
## Array element sizes:    4,    1,    1,    2,    2,    4,    4,    8,    8
## Array base offsets:    16,   16,   16,   16,   16,   16,   16,   16,   16
```

上述日志说明 

- Java引用占用4个字节（启用压缩引用）
- boolean/byte占用1个字节，char/short占用2个字节，int/float占用4个字节，double/long占用8个字节。
- 基础类型和引用作为数组元素呈现时占用相同的空间
- 数组基础offset(对象头+长度)是16，没有padding。

### 对象

使用例子查看对象的内存布局。

```java
public class JolCase01 {
  public static void main(String[] args) {
    System.out.println(ClassLayout.parseClass(A.class).toPrintable());
  }

  public static class A {
    long f;
  }
}
```

正如我们在前文介绍的，对象使用了8byte对齐，在不足的部分使用padding补齐。

```sh
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     N/A
  8   4        (object header: class)    N/A
 12   4        (alignment/padding gap)   
 16   8   long A.f                       N/A
Instance size: 24 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

### 数组

数组对象的长度没有放在数组类型中，JVM 在对象头中专门分配了一个区域用来存储数组长度。从本例中可以看到数组对象的长度信息。

```java
public class JolArray01 {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseInstance(new int[8]).toPrintable()); 
    }
}
```

从输出结果可以看到对象头中有 4 个字节的 array length 区域，存放数组长度。

```sh
[I object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf8000173
 12   4        (array length)            8
 16  32    int [I.<elements>             N/A
Instance size: 48 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

### 字段重排序

使用例子查看多字段对象的内存布局。

```java
public class JolCase02 {
      public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(A.class).toPrintable());
    }

    public static class A {
        boolean bo1;
        byte b1;
        char c1, c2;
        double d1, d2;
        float f1, f2;
        int i1, i2;
        long l1, l2;
        short s1, s2;
    }
}
```

正如我们在前文介绍的，对象使用了字段重排序，尽可能的填满内存。

```sh
info.victorchu.demos.jol.quickstart.JolCase02$A object internals:
OFF  SZ      TYPE DESCRIPTION               VALUE
  0   8           (object header: mark)     N/A
  8   4           (object header: class)    N/A
 12   4     float A.f1                      N/A
 16   8    double A.d1                      N/A
 24   8    double A.d2                      N/A
 32   8      long A.l1                      N/A
 40   8      long A.l2                      N/A
 48   4     float A.f2                      N/A
 52   4       int A.i1                      N/A
 56   4       int A.i2                      N/A
 60   2      char A.c1                      N/A
 62   2      char A.c2                      N/A
 64   2     short A.s1                      N/A
 66   2     short A.s2                      N/A
 68   1   boolean A.bo1                     N/A
 69   1      byte A.b1                      N/A
 70   2           (object alignment gap)    
Instance size: 72 bytes
Space losses: 0 bytes internal + 2 bytes external = 2 bytes total
```

### 继承字段顺序

使用例子查看继承对象的内存布局。

```java
public class JolCase03 {
     public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(C.class).toPrintable());
    }

    public static class A {
        int a;
    }

    public static class B extends A {
        int b;
    }

    public static class C extends B {
        int c;
    }
}
```

正如我们在前文介绍的，继承关系中，JVM会首先存放超类的字段。

```sh
info.victorchu.demos.jol.quickstart.JolCase03$C object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     N/A
  8   4        (object header: class)    N/A
 12   4    int A.a                       N/A
 16   4    int B.b                       N/A
 20   4    int C.c                       N/A
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

### 继承字段重排

```java
public class JolCase04 {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(C.class).toPrintable());
    }
    public static class A {
        long a;
    }
    public static class B extends A {
        int b;
    }
    public static class C extends B {
        long c;
        int d;
    }
}
```

如果子类首个成员变量是 long 或者 double 等 8 字节数据类型，而父类结束时没有 8 位对齐。会把子类的小于 8 字节的实例成员先排列，直到能 8 字节对齐。

```sh
info.victorchu.demos.jol.quickstart.JolCase04$C object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     N/A
  8   4        (object header: class)    N/A
 12   4        (alignment/padding gap)   
 16   8   long A.a                       N/A
 24   4    int B.b                       N/A
 28   4    int C.d                       N/A
 32   8   long C.c                       N/A
Instance size: 40 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

注意到上面对象头和字段中间存在 gap。在Java15后，对整个字段布局进行了大修，在这种情况下，会使用子类字段填充对象头。

```sh
// java 17
info.victorchu.demos.jol.quickstart.JolCase04$C object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     N/A
  8   4        (object header: class)    N/A
 12   4    int B.b                       N/A
 16   8   long A.a                       N/A
 24   8   long C.c                       N/A
 32   4    int C.d                       N/A
 36   4        (object alignment gap)    
Instance size: 40 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

### 类继承gap

考虑下面这段代码:

```java
public class JolCase05 {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(C.class).toPrintable());
    }
    public static class A {
        boolean a;
    }
    public static class B extends A {
        boolean b;
    }
    public static class C extends B {
        boolean c;
    }
}
```

由于A中的字段不足8byte，所以需要在类字段后面增加padding gap。

```sh
info.victorchu.demos.jol.quickstart.JolCase05$C object internals:
OFF  SZ      TYPE DESCRIPTION               VALUE
  0   8           (object header: mark)     N/A
  8   4           (object header: class)    N/A
 12   1   boolean A.a                       N/A
 13   3           (alignment/padding gap)   
 16   1   boolean B.b                       N/A
 17   3           (alignment/padding gap)   
 20   1   boolean C.c                       N/A
 21   3           (object alignment gap)    
Instance size: 24 bytes
Space losses: 6 bytes internal + 3 bytes external = 9 bytes total
```

这个问题在Java15后，也被优化，在这种情况下，不会出现类字段之间的gap。


```sh
info.victorchu.demos.jol.quickstart.JolCase05$C object internals:
OFF  SZ      TYPE DESCRIPTION               VALUE
  0   8           (object header: mark)     N/A
  8   4           (object header: class)    N/A
 12   1   boolean A.a                       N/A
 13   1   boolean B.b                       N/A
 14   1   boolean C.c                       N/A
 15   1           (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 1 bytes external = 1 bytes total
```

### 特殊类


```java
public class JolCase06 {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(Throwable.class).toPrintable());
        System.out.println(ClassLayout.parseClass(Class.class).toPrintable());
    }
}
```

在JDK 8及以下版本中，可以看到一些关于Throwable 类不合理的内存空隙。如果查看Throwable类的源码，可以看到 Throwable.backtrace 字段，这个字段在内存dump中找不到。 这是因为该字段保存的是虚拟机内部的数据，是不允许用户访问的。

```sh
java.lang.Throwable object internals:
OFF  SZ                            TYPE DESCRIPTION                      VALUE
  0   8                                 (object header: mark)            N/A
  8   4                                 (object header: class)           N/A
 12   4                                 (alignment/padding gap)          
 16   4                java.lang.String Throwable.detailMessage          N/A
 20   4             java.lang.Throwable Throwable.cause                  N/A
 24   4   java.lang.StackTraceElement[] Throwable.stackTrace             N/A
 28   4                  java.util.List Throwable.suppressedExceptions   N/A
Instance size: 32 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

但是在JDK9及其后续版本，这个字段是可见的。

```sh
java.lang.Throwable object internals:
OFF  SZ                            TYPE DESCRIPTION                      VALUE
  0   8                                 (object header: mark)            N/A
  8   4                                 (object header: class)           N/A
 12   4                java.lang.Object Throwable.backtrace              N/A
 16   4                java.lang.String Throwable.detailMessage          N/A
 20   4             java.lang.Throwable Throwable.cause                  N/A
 24   4   java.lang.StackTraceElement[] Throwable.stackTrace             N/A
 28   4                  java.util.List Throwable.suppressedExceptions   N/A
 32   4                             int Throwable.depth                  N/A
 36   4                                 (object alignment gap)           
Instance size: 40 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

Class类的实例字段中有很大的gap，同时检查Class的java代码也找不到对应的java字段。这些并不是不可见字段，而是虚拟机"注入"的元信息。

```sh
java.lang.Class object internals:
OFF  SZ                                              TYPE DESCRIPTION                    VALUE
  0   8                                                   (object header: mark)          N/A
  8   4                                                   (object header: class)         N/A
 12   4                     java.lang.reflect.Constructor Class.cachedConstructor        N/A
 16   4                                   java.lang.Class Class.newInstanceCallerCache   N/A
 20   4                                  java.lang.String Class.name                     N/A
 24   4                                  java.lang.Module Class.module                   N/A
 28   4                                                   (alignment/padding gap)        
 32   4                                  java.lang.String Class.packageName              N/A
 36   4                                   java.lang.Class Class.componentType            N/A
 40   4                       java.lang.ref.SoftReference Class.reflectionData           N/A
 44   4   sun.reflect.generics.repository.ClassRepository Class.genericInfo              N/A
 48   4                                java.lang.Object[] Class.enumConstants            N/A
 52   4                                     java.util.Map Class.enumConstantDirectory    N/A
 56   4                    java.lang.Class.AnnotationData Class.annotationData           N/A
 60   4             sun.reflect.annotation.AnnotationType Class.annotationType           N/A
 64   4                java.lang.ClassValue.ClassValueMap Class.classValueMap            N/A
 68  28                                                   (alignment/padding gap)        
 96   4                                               int Class.classRedefinedCount      N/A
100   4                                                   (object alignment gap)         
Instance size: 104 bytes
Space losses: 32 bytes internal + 4 bytes external = 36 bytes total
```

### `@Contended`

java使用`@sun.misc.Contended`(JDK9后注解改成`@jdk.internal.vm.annotation.Contended`)注解解决CPU伪共享问题。使用`@Contended`注解需要关闭jvm选项`-XX:-RestrictContended`。

```java
public class JolCase07 {
    public static void main(String[] args) {
        System.out.println(ClassLayout.parseClass(B.class).toPrintable());
    }
    public static class A {
        int a;
        int b;
        @Contended
        int c;
        int d;
    }
    public static class B extends A {
        int e;
        @sun.misc.Contended("first")
        int f;
        @sun.misc.Contended("first")
        int g;
        @sun.misc.Contended("last")
        int i;
        @sun.misc.Contended("last")
        int k;
    }
}
```

可以看到`@Contended`会在对象布局时产生128长度的padding gap(可以通过`-XX:ContendedPaddingWidth`设置gap长度)。且 f和g，i和k被放在了不同组中。

```sh
info.victorchu.demos.jol.quickstart.JolCase07$B object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     N/A
  8   4        (object header: class)    N/A
 12   4    int A.a                       N/A
 16   4    int A.b                       N/A
 20   4    int A.d                       N/A
 24 128        (alignment/padding gap)   
152   4    int A.c                       N/A
156 128        (alignment/padding gap)   
284   4    int B.e                       N/A
288 128        (alignment/padding gap)   
416   4    int B.f                       N/A
420   4    int B.g                       N/A
424 128        (alignment/padding gap)   
552   4    int B.i                       N/A
556   4    int B.k                       N/A
Instance size: 560 bytes
Space losses: 512 bytes internal + 0 bytes external = 512 bytes total
```

## MarkWord

对象头中的mark words 中会存放锁信息，它是实现轻量级锁和偏向锁的关键。在申请锁然后再释放锁的过程中，我们可以清晰的看到 mark words 值的变化。

### 偏向锁

下面的例子用来说明偏向锁的变化情况:

- JDK 9之前偏向锁在JVM启动5秒后才可以使用。因此，在JDK 8及以下版本中运行本例的时候需要增加JVM运行参数 `-XX:BiasedLockingStartupDelay=0`。
- 从 JDK 15 之后偏向锁不再是默认的锁，所以在JDK 15 中运行本例需要增加JVM运行参数：`-XX:+UseBiasedLocking=`。

```java
public class JolMarkWord01 {
     public static void main(String[] args) {
        final A a = new A();
        ClassLayout layout = ClassLayout.parseInstance(a);
        System.out.println("**** Fresh object");
        System.out.println(layout.toPrintable());
        synchronized (a) {
            System.out.println("**** With the lock");
            System.out.println(layout.toPrintable());
        }
        System.out.println("**** After the lock");
        System.out.println(layout.toPrintable());
    }
    public static class A {
        // no fields
    }
}
```

日志如下:

```sh
**** Fresh object
info.victorchu.demos.jol.quickstart.JolMarkWord01$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the lock
info.victorchu.demos.jol.quickstart.JolMarkWord01$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007ff91800c005 (biased: 0x0000001ffe460030; epoch: 0; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the lock
info.victorchu.demos.jol.quickstart.JolMarkWord01$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007ff91800c005 (biased: 0x0000001ffe460030; epoch: 0; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

从输出结果可以看出：

- mark word 的值发生了变化（也就是VALUE字段的值），状态从biasable变到biased。
- 偏向锁释放时，不会修改markword。

### 轻量级锁

轻量级锁的目的是，在没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。如果是 JDK 8 及以下版本直接运行本例即可，不需要增加任何JVM参数。如果是 JDK 9运行时需要添加参数：`-XX:-UseBiasedLocking`。

```java
public class JolMarkWord02 {
    public static void main(String[] args) {
        final A a = new A();
        ClassLayout layout = ClassLayout.parseInstance(a);

        System.out.println("**** Fresh object");
        System.out.println(layout.toPrintable());

        synchronized (a) {
            System.out.println("**** With the lock");
            System.out.println(layout.toPrintable());
        }

        System.out.println("**** After the lock");
        System.out.println(layout.toPrintable());
    }

    public static class A {
        // no fields
    }

}
```

日志如下:

```sh
**** Fresh object
info.victorchu.demos.jol.quickstart.JolMarkWord02$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the lock
info.victorchu.demos.jol.quickstart.JolMarkWord02$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fee771f9958 (thin lock: 0x00007fee771f9958)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the lock
info.victorchu.demos.jol.quickstart.JolMarkWord02$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

从输出结果的 mark word 字段值可以看到对象头的初始状态为 "non-biasable"。当持有对象的锁时状态变为 "thin lock"。释放锁之后对象头的状态又恢复到初始状态。

### 重量级锁

当虚拟机检测到多线程竞争时，虚拟机就委托操作系统来实现互斥，也就是使用操作系统互斥量来实现锁的操作。为了模拟多线程竞争锁的场景，增加了一个竞争线程。

```java
public class JolMarkWord03 {
    public static void main(String[] args) throws Exception {
        final A a = new A();

        ClassLayout layout = ClassLayout.parseInstance(a);

        System.out.println("**** Fresh object");
        System.out.println(layout.toPrintable());

        Thread t = new Thread(() -> {
            synchronized (a) {
                try {
                    TimeUnit.SECONDS.sleep(10);
                } catch (InterruptedException e) {
                    // Do nothing
                }
            }
        });

        t.start();

        TimeUnit.SECONDS.sleep(1);

        System.out.println("**** Before the lock");
        System.out.println(layout.toPrintable());

        synchronized (a) {
            System.out.println("**** With the lock");
            System.out.println(layout.toPrintable());
        }

        System.out.println("**** After the lock");
        System.out.println(layout.toPrintable());

        System.gc();

        System.out.println("**** After System.gc()");
        System.out.println(layout.toPrintable());
    }

    public static class A {
        // no fields
    }
}
```

运行日志如下: 

```sh
**** Fresh object
info.victorchu.demos.jol.quickstart.JolMarkWord03$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** Before the lock
info.victorchu.demos.jol.quickstart.JolMarkWord03$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fcf5c70a8e0 (thin lock: 0x00007fcf5c70a8e0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the lock
info.victorchu.demos.jol.quickstart.JolMarkWord03$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fcf540049ea (fat lock: 0x00007fcf540049ea)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the lock
info.victorchu.demos.jol.quickstart.JolMarkWord03$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fcf540049ea (fat lock: 0x00007fcf540049ea)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After System.gc()
info.victorchu.demos.jol.quickstart.JolMarkWord03$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000009 (non-biasable; age: 1)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

从输出结果可以看到，开始时对象头的状态是默认状态，当线程 t 持有对象锁后对象头中的锁信息变为轻量级锁。然后当主线程持有锁后，对象头的轻量级锁膨胀为重量级锁。膨胀后的锁状态在锁释放后会一直保持，只有当经过GC后才会恢复到初始状态。

在jdk15以后，markword中的重量级锁并不一定在GC后清除。当有足够多的monitor未被清理时，才会触发异步清理。

### HashCode

hash code 存放在 mark word中。

```java
public class JolMarkWord04 {
   
     public static void main(String[] args) {
        final A a = new A();
        ClassLayout layout = ClassLayout.parseInstance(a);
        System.out.println("**** Fresh object");
        System.out.println(layout.toPrintable());

        synchronized (a) {
            System.out.println("**** With the lock");
            System.out.println(layout.toPrintable());
        }
        
        System.out.println("**** After the lock");
        System.out.println(layout.toPrintable());

        System.out.println("hashCode: " + Integer.toHexString(a.hashCode()));
        System.out.println(layout.toPrintable());

        synchronized (a) {
            System.out.println("**** With the second lock");
            System.out.println(layout.toPrintable());
        }

        System.out.println("**** After the second lock");
        System.out.println(layout.toPrintable());
    }

    public static class A {
        // no fields
    }
}
```

从输出结果可以看到新创建的对象头中并没有生成 hash code 的值，当调用hashCode() 方法后才会生成 hash code 的值。且 生成的 hash code 值在对象的生命周期内不会再改变。即便被锁信息覆写，一旦锁被释放，已经计算好的hash code又会被会写。

```sh
**** Fresh object
info.victorchu.demos.jol.quickstart.JolMarkWord04$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the lock
info.victorchu.demos.jol.quickstart.JolMarkWord04$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007f9b6c00c005 (biased: 0x0000001fe6db0030; epoch: 0; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the lock
info.victorchu.demos.jol.quickstart.JolMarkWord04$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007f9b6c00c005 (biased: 0x0000001fe6db0030; epoch: 0; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

hashCode: 3fa77460
info.victorchu.demos.jol.quickstart.JolMarkWord04$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000003fa7746001 (hash: 0x3fa77460; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** With the second lock
info.victorchu.demos.jol.quickstart.JolMarkWord04$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007f9b71377950 (thin lock: 0x00007f9b71377950)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

**** After the second lock
info.victorchu.demos.jol.quickstart.JolMarkWord04$A object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000003fa7746001 (hash: 0x3fa77460; age: 0)
  8   4        (object header: class)    0xf800c105
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

## 高级功能

### 对象引用

JOL 支持查看对象的所有引用信息。

```java
public class JolAdvance01 {
    public static void main(String[] args) {
        ArrayList<Integer> al = new ArrayList<>();
        LinkedList<Integer> ll = new LinkedList<>();

        for (int i = 0; i < 1000; i++) {
            Integer io = i; // box once
            al.add(io);
            ll.add(io);
        }

        al.trimToSize();

        PrintWriter pw = new PrintWriter(System.out);
        pw.println(GraphLayout.parseInstance(al).toFootprint());
        pw.println(GraphLayout.parseInstance(ll).toFootprint());
        pw.println(GraphLayout.parseInstance(al, ll).toFootprint());
        pw.close();
    }
}
```
上面的例子通过 ArrayList 和 LinkedList 来展示对象的引用信息。也可以同时查看多个对象的引用信息，但是 JOL 会避免重复计数，即两个对象中相同的部分只统计一次。

```sh
java.util.ArrayList@330bedb4d footprint:
     COUNT       AVG       SUM   DESCRIPTION
         1      4016      4016   [Ljava.lang.Object;
      1000        16     16000   java.lang.Integer
         1        24        24   java.util.ArrayList
      1002               20040   (total)


java.util.LinkedList@7d70d1b1d footprint:
     COUNT       AVG       SUM   DESCRIPTION
      1000        16     16000   java.lang.Integer
         1        32        32   java.util.LinkedList
      1000        24     24000   java.util.LinkedList$Node
      2001               40032   (total)


java.util.ArrayList@330bedb4d, java.util.LinkedList@7d70d1b1d footprint:
     COUNT       AVG       SUM   DESCRIPTION
         1      4016      4016   [Ljava.lang.Object;
      1000        16     16000   java.lang.Integer
         1        24        24   java.util.ArrayList
         1        32        32   java.util.LinkedList
      1000        24     24000   java.util.LinkedList$Node
      2003               44072   (total)
```

### 对象的内存地址

为了触发GC，运行时堆大小要小于1G(`-Xmx256M -Xms256M`)，可以打开GC日志(`-XX:+PrintGCDetails`)方便对比。
```java
public class JolAdvance02 {
    public static void main(String[] args) {
        PrintWriter pw = new PrintWriter(System.out, true);
        long last = VM.current().addressOf(new Object());
        for (int l = 0; l < 1000 * 1000 * 1000; l++) {
            long current = VM.current().addressOf(new Object());
            long distance = Math.abs(current - last);
            if (distance > 4096) {
                pw.printf("Jumping from %x to %x (distance = %d bytes, %dK, %dM)%n",
                        last,
                        current,
                        distance,
                        distance / 1024,
                        distance / 1024 / 1024);
            }
            last = current;
        }
        pw.close();
    }
}
// Jumping from ffefffd8 to f5580000
```

从输出结果可以看到地址发生了较大距离的切换。结合GC日志,可以得知这是GC导致的。

### 对象晋升

每次 GC 之后存活对象的年龄都会增长，当增长到一定阈值后就会从年轻代晋升到老年代。我们可以通过JOL 观察对象的年龄和观察对象晋升的过程(主要是通过对象地址的变化来确定，因为每次 GC 都会改变对象的存储位置)。

为了触发GC，运行时堆大小要小于1G(`-Xmx256M -Xms256M`)，可以打开GC日志(`-XX:+PrintGCDetails`)方便对比。

```java
public class JolAdvance03 {
    static volatile Object sink;

    public static void main(String[] args) {
        PrintWriter pw = new PrintWriter(System.out, true);

        Object o = new Object();

        ClassLayout layout = ClassLayout.parseInstance(o);

        long lastAddr = VM.current().addressOf(o);
        pw.printf("*** Fresh object is at %x%n", lastAddr);
        System.out.println(layout.toPrintable());

        int moves = 0;
        for (int i = 0; i < 100000; i++) {
            long cur = VM.current().addressOf(o);
            if (cur != lastAddr) {
                moves++;
                pw.printf("*** Move %2d, object is at %x%n", moves, cur);
                System.out.println(layout.toPrintable());
                lastAddr = cur;
            }

            // make garbage
            for (int c = 0; c < 10000; c++) {
                sink = new Object();
            }
        }

        long finalAddr = VM.current().addressOf(o);
        pw.printf("*** Final object is at %x%n", finalAddr);
        System.out.println(layout.toPrintable());

        pw.close();
    }
}
```


```sh
*** Fresh object is at f567fe30
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0x200001ed
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

*** Move  1, object is at fd620178
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000009 (non-biasable; age: 1)
  8   4        (object header: class)    0x200001ed
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

*** Move  2, object is at feb18a80
java.lang.Object object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000011 (non-biasable; age: 2)
  8   4        (object header: class)    0x200001ed
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

从输出结果可以看到对象的地址更改多次，每次对象的地址都不同，同时可以看到 JOL 输出的对象头中的 age 会随着增加。

### GC对数组的影响

无论选择哪种 GC 数组中初始的元素都是按照索引顺序存储的。但是如果我们选择并行GC，那么当发生 GC 后数组的元素就是按照倒序存储(建议参数`-XX:ParallelGCThreads=1`，如果是多线程，结果是混乱的)。

```java
/**
 * @see https://bugs.openjdk.java.net/browse/JDK-8024394
 */
public class JolAdvance04 {
    public static void main(String[] args) {
        PrintWriter pw = new PrintWriter(System.out, true);

        Integer[] arr = new Integer[10];
        for (int i = 0; i < 10; i++) {
            arr[i] = i + 256; // boxing outside of Integer cache
        }

        String last = null;
        for (int c = 0; c < 100; c++) {
            String current = GraphLayout.parseInstance((Object) arr).toPrintable();

            if (last == null || !last.equalsIgnoreCase(current)) {
                pw.println(current);
                last = current;
            }
            // 显式触发GC
            System.gc();
        }

        pw.close();
    }
}
```

从输出结果可以看到第一次数组的元素是按照索引顺序存储的，后续都是倒序存储。从本例也可以看出，垃圾收集器也可以影响数组对象在内存中的存储结构。

```sh
[Ljava.lang.Integer;@330bedb4d object externals:
          ADDRESS       SIZE TYPE                 PATH                           VALUE
        71567fc40         56 [Ljava.lang.Integer;                                [256, 257, 258, 259, 260, 261, 262, 263, 264, 265]
        71567fc78         16 java.lang.Integer    [0]                            256
        71567fc88         16 java.lang.Integer    [1]                            257
        71567fc98         16 java.lang.Integer    [2]                            258
        71567fca8         16 java.lang.Integer    [3]                            259
        71567fcb8         16 java.lang.Integer    [4]                            260
        71567fcc8         16 java.lang.Integer    [5]                            261
        71567fcd8         16 java.lang.Integer    [6]                            262
        71567fce8         16 java.lang.Integer    [7]                            263
        71567fcf8         16 java.lang.Integer    [8]                            264
        71567fd08         16 java.lang.Integer    [9]                            265

Addresses are stable after 1 tries.

[Ljava.lang.Integer;@330bedb4d object externals:
          ADDRESS       SIZE TYPE                 PATH                           VALUE
        5c001a9a0         56 [Ljava.lang.Integer;                                [256, 257, 258, 259, 260, 261, 262, 263, 264, 265]
        5c001a9d8      12080 (something else)     (somewhere else)               (something else)
        5c001d908         16 java.lang.Integer    [9]                            265
        5c001d918         16 java.lang.Integer    [8]                            264
        5c001d928         16 java.lang.Integer    [7]                            263
        5c001d938         16 java.lang.Integer    [6]                            262
        5c001d948         16 java.lang.Integer    [5]                            261
        5c001d958         16 java.lang.Integer    [4]                            260
        5c001d968         16 java.lang.Integer    [3]                            259
        5c001d978         16 java.lang.Integer    [2]                            258
        5c001d988         16 java.lang.Integer    [1]                            257
        5c001d998         16 java.lang.Integer    [0]                            256

Addresses are stable after 1 tries.

[Ljava.lang.Integer;@330bedb4d object externals:
          ADDRESS       SIZE TYPE                 PATH                           VALUE
        5c001a950         56 [Ljava.lang.Integer;                                [256, 257, 258, 259, 260, 261, 262, 263, 264, 265]
        5c001a988       9728 (something else)     (somewhere else)               (something else)
        5c001cf88         16 java.lang.Integer    [9]                            265
        5c001cf98         16 java.lang.Integer    [8]                            264
        5c001cfa8         16 java.lang.Integer    [7]                            263
        5c001cfb8         16 java.lang.Integer    [6]                            262
        5c001cfc8         16 java.lang.Integer    [5]                            261
        5c001cfd8         16 java.lang.Integer    [4]                            260
        5c001cfe8         16 java.lang.Integer    [3]                            259
        5c001cff8         16 java.lang.Integer    [2]                            258
        5c001d008         16 java.lang.Integer    [1]                            257
        5c001d018         16 java.lang.Integer    [0]                            256

Addresses are stable after 1 tries.
```

## 原理

### Instrumentation

使用java.lang.instrument.Instrumentation.getObjectSize()方法，可以很方便的计算任何一个运行时对象的大小(Shadow heap size)，返回该对象本身在内存中的大小。不过，我们在代码中无法直接实例化它，需要在JVM启动时，通过指定代理的方式，让JVM来实例化它。其原理是通过反射，累加计算 Retained heap size。

### unsafe

java中的sun.misc.Unsafe类，有一个objectFieldOffset(Field f)方法，表示获取指定字段在所在实例中的起始地址偏移量，如此可以计算出指定的对象中每个字段的偏移量，值为最大的那个就是最后一个字段的首地址，加上该字段的实际大小，就能知道该对象整体的大小。


## 参考资料

- [1] [Java Objects Inside Out](https://shipilev.net/jvm/objects-inside-out/)
- [2] [JDK15.Field layout computation overhaul](https://bugs.openjdk.org/browse/JDK-8237767)

