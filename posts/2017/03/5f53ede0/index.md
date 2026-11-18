# Java多态-继承绑定

本文介绍了Java 中静态方法，属性和方法与类的绑定关系。

<!--more-->

## 属性与静态方法

Java中属性与静态方法和类绑定(静态绑定)。静态方法和属性是属于类的，调用的时候直接通过类名.方法名完成对，不需要继承机制及可以调用。如果子类里面定义了同名静态方法和属性，那么这时候父类的静态方法或属性称之为"隐藏"。如果你想要调用父类的静态方法和属性，直接通过父类名.方法或变量名完成。

## 方法绑定 

### 程序绑定的概念：

绑定指的是一个方法的调用与方法所在的类(方法主体)关联起来。对java来说，绑定分为静态绑定和动态绑定；或者叫做前期绑定和后期绑定。

### 静态绑定（早绑定 编译器绑定）：

在程序执行前方法已经被绑定，此时由编译器或其它连接程序实现。针对java可以理解为程序编译期的绑定；特别说明一点，java当中的方法只有final，static，private和构造方法是前期绑定

### 动态绑定（迟绑定 运行期绑定）：

后期绑定：在运行时根据具体对象的类型进行绑定。若一种语言实现了后期绑定，同时必须提供一些机制在运行期间判断对象的类型，并分别调用适当的方法。也就是说编译器此时依然不知道对象的类型，但方法调用机制能自己去调查，找到正确的方法主体。不同的语言对后期绑定的实现方法是有所区别的。可以这样认为：它们都要在对象中安插某些特殊类型的信息。

#### 动态绑定的过程：

* 虚拟机提取对象的实际类型的方法表
* 虚拟机搜索方法签名
* 调用方法

### 总结

了解三者的概念之后，我们发现java属于后期绑定。在java中，几乎所有的方法都是后期绑定，在运行时动态绑定方法属于子类还是基类。但也有特殊，针对static方法和final方法由于不能被继承，因此在编译时就可以确定他们的值，他们是属于前期绑定。特别说明的一点，private声明的方法和成员变量不能被子类继承，所有的private方法都被隐式的指定为final的\(由此我们知道：将方法声明为final类型，一是为了防止方法被覆盖，二是为了有效的关闭java中的动态绑定\)。java中的后期绑定是由JVM来实现的，我们不用去显式的声明它。

## 代码例子

### 一个动态绑定的例子

动态绑定的典型发生在父类和子类的转换声明之下：比如：`Parent p = new Children();`具体过程如下：

1. 编译器检查对象的声明类型和方法名。假设我们调用x.f(args)方法，并且x已经被声明为C类的对象，那么编译器会列举出C类中所有的名称为f的方法和从C类的超类继承过来的f方法
2. 接下来编译器检查方法调用中提供的参数类型。如果在所有名称为f 的方法中有一个参数类型和调用提供的参数类型最为匹配，那么就调用这个方法，这个过程叫做“重载解析”
3. 当程序运行并且使用动态绑定调用方法时，虚拟机必须调用同x所指向的对象的实际类型相匹配的方法版本。假设实际类型为D(C的子类)，如果D类定义了f(String)那么该方法被调用，否则就在D的超类中搜寻方法f(String),依次类推。

### 绑定验证demo

* 基类

```java
public class Base {
    public String name = "Base class";
    public static void staticMethod()
    {
        System.out.println("Base staticMethod()");
    }
    public void CommonMethod()
    {
        System.out.println("Base CommonMethod()");
    }
    public String getName() {  
        return name;  
    }  
}
```

* 衍生类

```java
public class Derive extends Base{
    public String name = "Derive class";
    public static void staticMethod()
    {
        System.out.println("Derive staticMethod()");
    }
    public void CommonMethod()
    {
        System.out.println("Derive CommonMethod()");
    }
    public String getName() {  
        return name;  
    }  
}
```

* 测试类

```java
public class testblind {
    //输出成员变量的值，验证其为前期绑定。
    public static void testFieldBind(Base base)
    {
        System.out.println(base.name);
    }
    //静态方法，验证其为前期绑定。
    public static void testStaticMethodBind(Base base)
    {
        base.staticMethod();
    }
    //调用非静态方法，验证其为后期绑定。
    public static void testNotStaticMethodBind(Base base)
    {
        base.CommonMethod();
    }
    public static void main(String[] args) {
        Derive d= new Derive();
        Base b = new Derive();
        testFieldBind(d);
        System.out.println(d.getName());
        testStaticMethodBind(d);
        testNotStaticMethodBind(d);
        
        testFieldBind(b);
        System.out.println(b.getName());
        testStaticMethodBind(b);
        testNotStaticMethodBind(b);
    }

}
```

* 输出

```sh
Base class
Derive class
Base staticMethod()
Derive CommonMethod()
Base class
Derive class
Base staticMethod()
Derive CommonMethod()
```

