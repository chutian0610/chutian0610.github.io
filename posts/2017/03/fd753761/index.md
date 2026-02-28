# Java final 关键字

Java `final`关键字有以下几个使用场景:

* 修饰类，代表不可以继承扩展
* 修饰变量，代表变量不可以修改
* 修饰方法，代表方法不可以重写

<!--more-->

## final修饰变量

### 常数

1. 编译期间的常数,例如: final int i1 = 0;
2. 运行期间初始化的一个值，但不希望它发生变化,例如: final int i2 = (int)(Math.random()*20);

象上面这样，对基本数据类型使用final，使得其值不变。但是如果对一个对象句柄使用final，含义就不一样了。final会将句柄值变成一个常数，即句柄指向的对象不能发生变化，但是对象本身的只是可变的。需要注意的是：声明final 句柄时必须将句柄初始化到一个对象。

### 空白final

java 允许创建空白的final类变量，如: final int i1; 但是，i1 需要在构造器中初始化，这并不违反final 必须要有值的要求。要么在定义时初始化，要么在构造器中初始化。这样可以保证final 字段在使用时获得正确的初始化。而且使用构造器初始化，带来了很大的灵活性。

### final 方法参数

我们可以将方法参数设成 final 属性,方法是在方法参数列表中对它们进行适当的声明,如: public void test (final String str)。这意味着在一个方法的内部,我们不能改变方法参数句柄指向的东西。

### final 局部变量

局部内部类(定义在方法中的内部类)访问方法内的局部变量时，该变量必须是final修饰的。对jvm有了解的话，可以知道，局部变量的生命周期与局部内部类的对象的生命周期是不一致的。对于方法调用f(i)当方法f运行结束后,局部变量i就已死亡了,不存在了.但局部内部类对象还可能一直存在(对象存活取决于jvm何时回收)，特别是对象逃逸时(如果对象呗外部引用，那么会一直存活),它不会随着方法运行结束立刻死亡.这时出现了一个问题，局部内部类对象要访问一个已不存在的局部变量。

为了解决这个问题，当变量是final时,将final局部变量”复制”一份,复制品直接作为局部内部类中的数据成员。当变量是final时,若是基本数据类型,由于其值不变,其复制品与原始的量是一样.当变量是final时,若是引用类型,由于其引用值不变(即:永远指向同一个对象),因而:其复制品与原始的引用变量一样,永远指向同一个对象[^1]。

### 例子

```java

class Value { 
    int i = 1; 
    } 

public class TestFinal {
    final int i0 = 8;
    final int i1; 
    
    Value v1 = new Value(); 
    final Value v2 = new Value(); 
    static final Value v3 = new Value(); 
    final Value v4; 
    
    final int[] a = { 1, 2, 3, 4, 5, 6 }; 
    
    public TestFinal(){
        i1 = 9;
        v4 = new Value(); 
        v4.i = 9;
    }
     
    final int i4 = (int)(Math.random()*20); 
    static final int i5 = (int)(Math.random()*20); 
    
    public void print(final String id) { 
         //! id = "hello"; //Error: can't change handle
         System.out.println( 
           id + ": " + "i4 = " + i4 +  
           ", i5 = " + i5); 
        }  

    public static void main(String[] args) {
        TestFinal fd1 = new TestFinal(); 
        //! fd1.i0++; // Error: can't change value 
        //! fd1.i1++; // Error: can't change value
         fd1.v1 = new Value(); // OK -- not final 
         fd1.v2.i++; // Object isn't constant! 
         
         for(int i = 0; i < fd1.a.length; i++) 
           fd1.a[i]++; // Object isn't constant! 
         
         //! fd1.v2 = new Value(); // Error: Can't change handle
         //! fd1.v3 = new Value(); // Error: Can't change handle
         //! fd1.a = new int[3]; // Error: Can't change handle

         fd1.print("fd1"); 
         System.out.println("Creating new FinalData"); 

         TestFinal fd2 = new TestFinal(); 
         fd1.print("fd1"); 
         fd2.print("fd2");
         fd2.print(null);
    }

}
/*
 * ===================== OUTPUT =========================
 * fd1: i4 = 6, i5 = 10
 * Creating new FinalData
 * fd1: i4 = 6, i5 = 10
 * fd2: i4 = 15, i5 = 10
 * null: i4 = 15, i5 = 10
 */
```

## final 修饰方法

final + 方法,一般表示方法可被子类继承，但不能被重写覆盖。

> 类内所有 private 方法都自动成为 final。由于我们不能访问一个 private 方法,所以它绝对不会被其他方法覆盖(若强行这样做,编译器会给出错误提示)。可为一个 private 方法添加 final 指示符,但却不能为那个方法提供任何额外的含义。 

## final 修饰类

final + Class,表示此类不能被继承。

## final 与线程安全

当构造函数完成时，对象被认为是完全初始化的。在该对象完全初始化之后只能看到对象引用的线程可以保证看到该对象final字段的正确初始化值。

final 字段的使用模型很简单：在该对象的构造函数中设置对象的final字段; 并且在对象的构造函数完成之前，不要在另一个线程可以看到的地方写入对正在构造的对象的引用。如果遵循此操作，那么当另一个线程看到该对象时，该线程将始终看到该对象的final字段的正确构造版本。

```java
class FinalFieldExample {
    final int x;
    int y;
    static FinalFieldExample f;

    public FinalFieldExample（）{
        x = 3;
        y = 4;
    }

    static void writer（）{
        f = new FinalFieldExample（）;
    }

    static void reader（）{
        if（f！= null）{
            int i = fx; //保证看到3
            int j = fy; //可以看到0
        }
    }
}
```

该类FinalFieldExample有一个 final int字段x和一个非final int 字段y。一个线程可能执行该方法writer，另一个线程可能执行该方法reader。

因为该writer方法f 在对象的构造函数完成后写入，所以该reader方法将保证看到正确初始化的值f.x：它将读取该值3。但是，f.y不是 final; reader不保证该方法可以看到它的值4。

出现这种情况的原因是构造函数发生了指令重排序。

创建一个对象要分为三个步骤：

* 分配内存空间
* 初始化对象
* 将内存空间的地址赋值给对应的引用

但是由于指令重排序的问题，步骤 2 和步骤 3 是可能发生重排序的，如下：

* 分配内存空间
* 将内存空间的地址赋值给对应的引用
* 初始化对象

指令重排序在后面关于并发的文章中会有更详细的阐述，此处就不展开了。

[^1]: lambda 表达式中可以通过使用final解决的局部变量生命周期问题。

