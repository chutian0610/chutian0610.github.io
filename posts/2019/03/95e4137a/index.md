# Java8 lambda 底层实现原理

之前有听说过 java 8 的 lambda 实现不再是java 1.5的匿名内部类语法糖。今天刚好有时间看看:

```java
public class App {

    @FunctionalInterface
    public interface LambdaDemo{
        public void runLambda();
    }
    public static void doSomething(LambdaDemo demo){
        demo.runLambda();
    }

    public static void main(String[] args) {
        doSomething(()->System.out.println("hello world!"));
    }
}
```

<!--more-->

经过编译后，生成了两个 class 文件:

```sh
$ ls
App.class
'App$LambdaDemo.class'
```

显然，`App$LambdaDemo.class`是interface lambdaDemo的 class 文件。我们可以用命令查看下:

```sh
$ javap -c  -v App\$LambdaDemo
警告: 二进制文件App$LambdaDemo包含cn.victor.study.App$LambdaDemo
Classfile /Users/chutian/IdeaProjects/demos/target/classes/cn/victor/study/App$LambdaDemo.class
  Last modified 2019-3-2; size 283 bytes
  MD5 checksum af7c9127a2a697374979d7c62f0fb056
  Compiled from "App.java"
public interface cn.victor.study.App$LambdaDemo
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_INTERFACE, ACC_ABSTRACT
Constant pool:
   #1 = Class              #10            // cn/victor/study/App$LambdaDemo
   #2 = Class              #13            // java/lang/Object
   #3 = Utf8               runLambda
   #4 = Utf8               ()V
   #5 = Utf8               SourceFile
   #6 = Utf8               App.java
   #7 = Utf8               RuntimeVisibleAnnotations
   #8 = Utf8               Ljava/lang/FunctionalInterface;
   #9 = Class              #14            // cn/victor/study/App
  #10 = Utf8               cn/victor/study/App$LambdaDemo
  #11 = Utf8               LambdaDemo
  #12 = Utf8               InnerClasses
  #13 = Utf8               java/lang/Object
  #14 = Utf8               cn/victor/study/App
{
  public abstract void runLambda();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_ABSTRACT
}
SourceFile: "App.java"
RuntimeVisibleAnnotations:
  0: #8()
InnerClasses:
     public static #11= #1 of #9; //LambdaDemo=class cn/victor/study/App$LambdaDemo of class cn/victor/study/App
```

接下来看看lambda 究竟是如何实现的:

```sh
$ javap -c -v App
警告: 二进制文件App包含cn.victor.study.App
Classfile /Users/chutian/IdeaProjects/demos/target/classes/cn/victor/study/App.class
  Last modified 2019-3-2; size 1353 bytes
  MD5 checksum 7c3e6a94f67374296a68af43fa788f7b
  Compiled from "App.java"
public class cn.victor.study.App
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #9.#31         // java/lang/Object."<init>":()V
   #2 = InterfaceMethodref #10.#32        // cn/victor/study/App$LambdaDemo.runLambda:()V
   #3 = InvokeDynamic      #0:#37         // #0:runLambda:()Lcn/victor/study/App$LambdaDemo;
   #4 = Methodref          #8.#38         // cn/victor/study/App.doSomething:(Lcn/victor/study/App$LambdaDemo;)V
   #5 = Fieldref           #39.#40        // java/lang/System.out:Ljava/io/PrintStream;
   #6 = String             #41            // hello world!
   #7 = Methodref          #42.#43        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #8 = Class              #44            // cn/victor/study/App
   #9 = Class              #45            // java/lang/Object
  #10 = Class              #46            // cn/victor/study/App$LambdaDemo
  #11 = Utf8               LambdaDemo
  #12 = Utf8               InnerClasses
  #13 = Utf8               <init>
  #14 = Utf8               ()V
  #15 = Utf8               Code
  #16 = Utf8               LineNumberTable
  #17 = Utf8               LocalVariableTable
  #18 = Utf8               this
  #19 = Utf8               Lcn/victor/study/App;
  #20 = Utf8               doSomething
  #21 = Utf8               (Lcn/victor/study/App$LambdaDemo;)V
  #22 = Utf8               demo
  #23 = Utf8               Lcn/victor/study/App$LambdaDemo;
  #24 = Utf8               main
  #25 = Utf8               ([Ljava/lang/String;)V
  #26 = Utf8               args
  #27 = Utf8               [Ljava/lang/String;
  #28 = Utf8               lambda$main$0
  #29 = Utf8               SourceFile
  #30 = Utf8               App.java
  #31 = NameAndType        #13:#14        // "<init>":()V
  #32 = NameAndType        #47:#14        // runLambda:()V
  #33 = Utf8               BootstrapMethods
  #34 = MethodHandle       #6:#48         // invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #35 = MethodType         #14            //  ()V
  #36 = MethodHandle       #6:#49         // invokestatic cn/victor/study/App.lambda$main$0:()V
  #37 = NameAndType        #47:#50        // runLambda:()Lcn/victor/study/App$LambdaDemo;
  #38 = NameAndType        #20:#21        // doSomething:(Lcn/victor/study/App$LambdaDemo;)V
  #39 = Class              #51            // java/lang/System
  #40 = NameAndType        #52:#53        // out:Ljava/io/PrintStream;
  #41 = Utf8               hello world!
  #42 = Class              #54            // java/io/PrintStream
  #43 = NameAndType        #55:#56        // println:(Ljava/lang/String;)V
  #44 = Utf8               cn/victor/study/App
  #45 = Utf8               java/lang/Object
  #46 = Utf8               cn/victor/study/App$LambdaDemo
  #47 = Utf8               runLambda
  #48 = Methodref          #57.#58        // java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #49 = Methodref          #8.#59         // cn/victor/study/App.lambda$main$0:()V
  #50 = Utf8               ()Lcn/victor/study/App$LambdaDemo;
  #51 = Utf8               java/lang/System
  #52 = Utf8               out
  #53 = Utf8               Ljava/io/PrintStream;
  #54 = Utf8               java/io/PrintStream
  #55 = Utf8               println
  #56 = Utf8               (Ljava/lang/String;)V
  #57 = Class              #60            // java/lang/invoke/LambdaMetafactory
  #58 = NameAndType        #61:#64        // metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #59 = NameAndType        #28:#14        // lambda$main$0:()V
  #60 = Utf8               java/lang/invoke/LambdaMetafactory
  #61 = Utf8               metafactory
  #62 = Class              #66            // java/lang/invoke/MethodHandles$Lookup
  #63 = Utf8               Lookup
  #64 = Utf8               (Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
  #65 = Class              #67            // java/lang/invoke/MethodHandles
  #66 = Utf8               java/lang/invoke/MethodHandles$Lookup
  #67 = Utf8               java/lang/invoke/MethodHandles
{
  public cn.victor.study.App();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcn/victor/study/App;

  public static void doSomething(cn.victor.study.App$LambdaDemo);
    descriptor: (Lcn/victor/study/App$LambdaDemo;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokeinterface #2,  1            // InterfaceMethod cn/victor/study/App$LambdaDemo.runLambda:()V
         6: return
      LineNumberTable:
        line 14: 0
        line 15: 6
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       7     0  demo   Lcn/victor/study/App$LambdaDemo;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=1, args_size=1
         0: invokedynamic #3,  0              // InvokeDynamic #0:runLambda:()Lcn/victor/study/App$LambdaDemo;
         5: invokestatic  #4                  // Method doSomething:(Lcn/victor/study/App$LambdaDemo;)V
         8: return
      LineNumberTable:
        line 18: 0
        line 19: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;

  private static void lambda$main$0();
    descriptor: ()V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #6                  // String hello world!
         5: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 18: 0
}
SourceFile: "App.java"
InnerClasses:
     public static #11= #10 of #8; //LambdaDemo=class cn/victor/study/App$LambdaDemo of class cn/victor/study/App
     public static final #63= #62 of #65; //Lookup=class java/lang/invoke/MethodHandles$Lookup of class java/lang/invoke/MethodHandles
BootstrapMethods:
  0: #34 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #35 ()V
      #36 invokestatic cn/victor/study/App.lambda$main$0:()V
      #35 ()V

```

注意看 main 方法的字节码，方法的 code 属性里:

```java
0: invokedynamic #3,  0              // InvokeDynamic #0:runLambda:()Lcn/victor/study/App$LambdaDemo;
5: invokestatic  #4                  // Method doSomething:(Lcn/victor/study/App$LambdaDemo;)V
8: return
```

lambda 的实现是通过 invokedynamic 指令来实现的。java虚拟机里调用方法的字节码指令有5种：

* invokestatic　　//调用静态方法
* invokespecial　　//调用私有方法、实例构造器方法、父类方法
* invokevirtual　　//调用实例方法
* invokeinterface　　//调用接口方法，会在运行时再确定一个实现此接口的对象
* invokedynamic　　//先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法，在此之前的4条调用指令，分派逻辑是固化在java虚拟机内部的，而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。

invokedynamic指令是在jvm7中新增的，invokedynamic出现的位置代表一个动态调用点 invokedynamic指令后面会跟一个指向常量池的调用点限定符，这个限定符会被解析为一个动态调用点。调用点限定符的符号引用为CONSTANT_InvokeDynamic_info结构。

```java
CONSTANT_InvokeDynamic_info{  
    u1 tag;  
    u2 bootstrap_method_attr_index;
    u2 name_and_type_index;  
}
```

我们再看`invokedynamic #3`中#3指向的线程池常量:

```java
#3 = InvokeDynamic      #0:#37         // #0:runLambda:()Lcn/victor/study/App$LambdaDemo;

// CONSTANT_NameAndType_info: 对一个字段或方法的部分符号引用
#37 = NameAndType        #47:#50        // runLambda:()Lcn/victor/study/App$LambdaDemo;
//#37 指向的是runLambda()方法。

// #0 是在字节码BootstrapMethods中，JVM规范规定，如果类的常量池中存在CONSTANT_InvokeDynamic_info的话，那么attributes表中就必定有且仅有一个BootstrapMethods属性。
BootstrapMethods:
  0: #34 invokestatic java/lang/invoke/LambdaMetafactory.metafactory:(Ljava/lang/invoke/MethodHandles$Lookup;Ljava/lang/String;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodType;Ljava/lang/invoke/MethodHandle;Ljava/lang/invoke/MethodType;)Ljava/lang/invoke/CallSite;
    Method arguments:
      #35 ()V
      #36 invokestatic cn/victor/study/App.lambda$main$0:()V
      #35 ()V

// 常量池#36:

#36 = MethodHandle       #6:#49         // invokestatic cn/victor/study/
#49 = Methodref          #8.#59         // cn/victor/study/App.lambda$main$0:()V
#59 = NameAndType        #28:#14        // lambda$main$0:()V

private static void lambda$main$0();
    descriptor: ()V
    flags: ACC_PRIVATE, ACC_STATIC, ACC_SYNTHETIC
    Code:
      stack=2, locals=0, args_size=0
         0: getstatic     #5                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #6                  // String hello world!
         5: invokevirtual #7                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 18: 0
}
```

CONSTANT_MethodHandle_info结构包含两项信息，其结构如下所示：

```java
CONSTANT_MethodHandle_info {
    u1 tag;
    u1 reference_kind;
    u2 reference_index;
}
```

常量池 #36 作为 invokeDynamic 的参数传入的是 lambda 函数的实现逻辑。根据BootstrapMethods对应的#34可以找到此处lambda InvokeDynamic指令对应的引导方法是LambdaMetafactory.metafactory,其返还一个CallSite。

```java
public static CallSite metafactory(MethodHandles.Lookup caller,
                                       String invokedName,
                                       MethodType invokedType,
                                       MethodType samMethodType,
                                       MethodHandle implMethod,
                                       MethodType instantiatedMethodType)
            throws LambdaConversionException {
        AbstractValidatingLambdaMetafactory mf;
        mf = new InnerClassLambdaMetafactory(caller, invokedType,
                                             invokedName, samMethodType,
                                             implMethod, instantiatedMethodType,
                                             false, EMPTY_CLASS_ARRAY, EMPTY_MT_ARRAY);
        mf.validateMetafactoryArgs();
        return mf.buildCallSite();
    }
```

到此处，基本了解了 lambda 表达式的底层实现，是基于 JVM  invokedynamic指令实现的。在后续博客中，会继续深入。

