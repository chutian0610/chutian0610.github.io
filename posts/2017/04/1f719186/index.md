# Java 对象初始化顺序

本文介绍 Java 对象初始化时，各个部分的启动顺序。
<!--more-->

## 对象初始化

- java 对象初始化时，会先加载对应的类,随后加载其基类(如果存在基类);
- 然后是类的`cinit()`,顺序是父类的类构造器clinit -> 子类的类构造器clinit。
  - clinit方法是由编译器自动收集类中的所有类变量的赋值动作和静态语句块中的语句合并产生的，编译器收集的顺序是由语句在源文件中出现的顺序所决定的，静态语句块只能访问到定义在静态语句块之前的变量，定义在它之后的变量，在前面的静态语句块可以赋值，但是不能访问。
  - 虚拟机会保证一个类的类构造器clinit>在多线程环境中被正确的加锁、同步，如果多个线程同时去初始化一个类，那么只会有一个线程去执行这个类的类构造器clinit，其他线程都需要阻塞等待，直到活动线程执行clinit方法完毕。特别需要注意的是，在这种情形下，其他线程虽然会被阻塞，但如果执行clinit方法的那条线程退出后，其他线程在唤醒之后不会再次进入/执行clinit方法，因为 在同一个类加载器下，一个类型只会被初始化一次。
- 然后对象实例初始化，实例初始化的顺序如下: 父类的实例初始化 -> 子类的实例初始化。实例初始化中的内容是:
  - 属性得到初值：基本值类型为默认值;对象句柄为null; 
    - 定义为final非静态基本数据类型的成员变量此时也会被初始化, 注意Integer等包装类不行；
    - 有且只有定义为final非静态的String成员变量，采用的“=”赋值初始化会被执行(非new)。
  - 属性定义时的初始化\(例如`int i =1`\)和代码块，的执行顺序取决于它们的代码位置顺序。
  - 构造函数;

## 示例

```java
package info.victorchu.cinit;

public class Parent {
    static Sample staticSample = new Sample("Parent:静态成员staticSample初始化");
    static {
        System.out.println("Parent:类static块执行");
        Sample.init(staticSample,"Parent:类static块>>静态成员staticSample初始化");
    }

    {
        System.out.println("Parent:对象非静态块1执行");
    }
    Sample sampleField1 = new Sample("Parent:普通成员sampleField1初始化");
    final Sample  finalSample = new Sample("Parent:final成员finalSample初始化");

    Parent() {
        sampleField1 = new Sample("Parent:构造函数>>普通成员sampleField1初始化");
        viewValues();
        System.out.println("Parent:构造函数被调用");
    }

    //调用子类的override函数，访问子类未初始化的非静态成员变量
    public void viewValues() {

    }
}
public class Child extends Parent{
    static Sample staticChildSample = new Sample("Child:静态成员staticChildSample初始化");
    static {
        System.out.println("Child:类static块执行");
        Sample.init(staticChildSample,"Child:类static块>>静态成员staticChildSample初始化");
    }
    final Sample finalChildSample = new Sample("Child:final成员finalChildSample初始化");
    Child() {
        System.out.println("Child:构造函数被调用");
    }

    private int derive0 = 888;
    final private int derive1 = 888;
    final private Integer derive2 = 888;
    final private String derive3 = new String("Hello World");
    final private String derive4 = "Hello World";

    @Override
    public void viewValues() {
        System.out.println("子类成员变量derive0 = " + derive0);
        System.out.println("子类成员变量derive1 = " + derive1);
        System.out.println("子类成员变量derive2 = " + derive2);
        System.out.println("子类成员变量derive3 = " + derive3);
        System.out.println("子类成员变量derive4 = " + derive4);
        System.out.println("子类成员变量finalChildSample = " + finalChildSample);
    }
}
public class ClassInitOrder {
    public static void main(String[] args) {
        Parent parent = new Child();
    }
}
class Sample {
    String s;
    Sample(String s) {
        this.s = s;
        System.out.println(s);
    }
    Sample(String s,String old) {
        this.s = s;
        System.out.println(s+"->"+old);
    }
    static Sample init(Sample s,String str) {
        if(s == null){
            return new Sample(str);
        }else {
            return new Sample(str,s.toString());
        }
    }
    @Override
    public String toString() {
        return this.s;
    }
}
```

输出结果如下:

```sh
Parent:静态成员staticSample初始化
Parent:类static块执行
Parent:类static块>>静态成员staticSample初始化->Parent:静态成员staticSample初始化
Child:静态成员staticChildSample初始化
Child:类static块执行
Child:类static块>>静态成员staticChildSample初始化->Child:静态成员staticChildSample初始化
Parent:对象非静态块1执行
Parent:普通成员sampleField1初始化
Parent:final成员finalSample初始化
Parent:构造函数>>普通成员sampleField1初始化
子类成员变量derive0 = 0
子类成员变量derive1 = 888
子类成员变量derive2 = null
子类成员变量derive3 = null
子类成员变量derive4 = Hello World
子类成员变量finalChildSample = null
Parent:构造函数被调用
Child:final成员finalChildSample初始化
Child:构造函数被调用
```

## Static 特例

但是上述情况存在特例，例如:

```java
public class ClassStaticInit {
    
    public static void main(String[] args) {
        staticFunction();
    }

    static ClassStaticInit st = new ClassStaticInit();

    static {   //静态代码块
        System.out.println("1");
    }

    {       // 实例代码块
        System.out.println("2");
    }

    ClassStaticInit() {    // 实例构造器
        System.out.println("3");
        System.out.println("a=" + a + ",b=" + b);
    }

    public static void staticFunction() {   // 静态方法
        System.out.println("4");
    }

    int a = 110;    // 实例变量
    static int b = 112;     // 静态变量
}
/* Output: 
        2
        3
        a=110,b=0
        1
        4
 */
```

在实例化上述程序中的st变量时，实际上是把实例初始化嵌入到了静态初始化流程中，并且在上面的程序中，嵌入到了静态初始化的起始位置。这就导致了实例初始化完全发生在静态初始化之前，当然，这也是导致a为110b为0的原因。

