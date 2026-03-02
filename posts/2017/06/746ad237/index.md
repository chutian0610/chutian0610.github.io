# 反编译分析Java枚举类型的实现

假设我们正在编写一个剪刀-布-石头游戏。我们可以使用三个任意整数（例如，0，1，2或88，128，168），三个字符串（"剪刀"，"布"，"石头"）来表示三个手势。主要缺点是我们需要检查程序中其他不可行的值（例如 3、"Rock"等）以确保正确性。

枚举是一种特殊类型，它为程序中的常量提供类型安全的实现。换句话说，我们可以声明一个类型的变量，它只接受预先定义的值。

<!--more-->

## Enum JDK 源码

```java
package java.lang;

import java.io.Serializable;
import java.io.IOException;
import java.io.InvalidObjectException;
import java.io.ObjectInputStream;
import java.io.ObjectStreamException;

public abstract class Enum<E extends Enum<E>>
        implements Comparable<E>, Serializable {
    
    // enum 实例的名称
    private final String name;
    public final String name() {
        return name;
    }

    // enum 实例的序数
    private final int ordinal;
    public final int ordinal() {
        return ordinal;
    }

    /**
     * 独占的构造器. 开发者不能使用这个构造器.
     * 使用于编译器生成的代码.
     */
    protected Enum(String name, int ordinal) {
        this.name = name;
        this.ordinal = ordinal;
    }

    public String toString() {
        return name;
    }

    public final boolean equals(Object other) {
        return this==other;
    }

    public final int hashCode() {
        return super.hashCode();
    }

    // 枚举对象不能克隆
    protected final Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException();
    }

    // Enum实现了Comparable接口，可以进行比较
    public final int compareTo(E o) {
        Enum<?> other = (Enum<?>)o;
        Enum<E> self = this;
        // 默认情况下，只有同类型的enum才进行比较（原因见后文），要实现不同类型的enum之间的比较，只能复写compareTo方法。
        if (self.getClass() != other.getClass() && self.getDeclaringClass() != other.getDeclaringClass())
            throw new ClassCastException();
        return self.ordinal - other.ordinal;
    }

    @SuppressWarnings("unchecked")
    public final Class<E> getDeclaringClass() {
        Class<?> clazz = getClass();
        Class<?> zuper = clazz.getSuperclass();
        return (zuper == Enum.class) ? (Class<E>)clazz : (Class<E>)zuper;
    }

    public static <T extends Enum<T>> T valueOf(Class<T> enumType,
                                                String name) {
        T result = enumType.enumConstantDirectory().get(name);
        if (result != null)
            return result;
        if (name == null)
            throw new NullPointerException("Name is null");
        throw new IllegalArgumentException(
            "No enum constant " + enumType.getCanonicalName() + "." + name);
    }

    /**
     * enum classes cannot have finalize methods.
     */
    protected final void finalize() { }

    // 枚举对象不能序列化和反序列化
    private void readObject(ObjectInputStream in) throws IOException,
        ClassNotFoundException {
        throw new InvalidObjectException("can't deserialize enum");
    }

    private void readObjectNoData() throws ObjectStreamException {
        throw new InvalidObjectException("can't deserialize enum");
    }
}
```

抽象类不能被实例化，所以我们在java程序中不能使用new关键字来声明一个Enum，如果想要定义可以使用这样的语法：

```java
enum enumName{
    value1,value2
    method1(){}
    method2(){}
}
```

注意，此处的enum是抽象类，但是无法通过编程来继承.

```java
public class TestEnumExtend extends Enum{
}
```

编译文件会报错:

```java
$ javac TestEnumExtend.java
TestEnumExtend.java:1: 错误: 类无法直接扩展 java.lang.Enum
public class TestEnumExtend extends Enum{
       ^
1 个错误
```

## 怎么理解`<E extends Enum<E>>`

`Enum<E extends Enum<E>>`就是一个Enum只接受一个Enum或者他的子类作为参数。相当于把一个子类或者自己当成参数，传入到自身，引起一些特别的语法效果。

```java
enum Color{
    RED,GREEN,YELLOW
}
enum Season{
    SPRING,SUMMER,WINTER
}
```

因为枚举类型的默认的序号都是从零开始的，显然 Color.RED.ordinal()和Season.SPRING.ordinal()的值都是0.

注意一下 `compareTo`方法，首先我们认为Enum的定义中没有使用`Enum<E extends Enum<E>>`，那么compareTo方法就要这样定义（因为没有使用泛型，所以就要使用Object，这也是Java中很多方法常用的方式）：

```java
// 不使用泛型
public final int compareTo(Object o) 
// 使用泛型
public final int compareTo(E o) {
  Enum other = (Enum)o;
  Enum self = this;
  if (self.getClass() != other.getClass() && 
      self.getDeclaringClass() != other.getDeclaringClass())
      throw new ClassCastException();
  return self.ordinal - other.ordinal;
}
```

如果在compareTo方法中不做任何处理的话，那么 Color.RED和Season.SPRING相等的（因为Season.SPRING的序号和Color.RED的序号都是 0 ）。但是，很明显， Color.RED和Season.SPRING并不相等。

但是Java使用`Enum<E extends Enum<E>>`声明Enum，并且在compareTo的中使用E作为参数来避免了这种问题。 以上两个条件限制Color.RED只能和Color定义出来的枚举进行比较，当我们试图使用`Color.RED.compareTo(Season.SPRING);` 这样的代码是，会报出这样的错误：

```java
The method compareTo(Color) in the type Enum<Color> is not applicable for the arguments (Season)
```

## 反编译

```java
public enum Fruit {  
    APPLE, PEAR, PEACH, ORANGE;  
}
// ------------------------ decompile ------------------------
// $ javac Fruit.java
// $ javap Fruit
// Compiled from "Fruit.java"
public final class Fruit extends java.lang.Enum<Fruit> {
  // 注意此处的class Fruit为final
  public static final Fruit APPLE;
  public static final Fruit PEAR;
  public static final Fruit PEACH;
  public static final Fruit ORANGE;
  public static Fruit[] values();
  public static Fruit valueOf(java.lang.String);
  static {};
}
```

可以看到，实际上在经过编译器编译后生成了一个 Fruit 类，该类继承自 Enum 类，且是 final 的。从这一点来看，Java 中的枚举类型似乎就是一个语法糖。

每一个枚举常量都对应类中的一个 public static final 的实例，这些实例的初始化应该是在 static {} 语句块中进行的。因为枚举常量都是 final 的，因而一旦创建之后就不能进行更改了。
此外，Fruit 类还实现了 values() 和 valueOf() 这两个静态方法。

```java
// Decompiled by Jad, $ jad -sjava Fruit.class
public final class Fruit extends Enum{

    public static Fruit[] values(){
        return (Fruit[])$VALUES.clone();
    }

    public static Fruit valueOf(String s){
        return (Fruit)Enum.valueOf(Fruit, s);
    }

    private Fruit(String s, int i){
        super(s, i);
    }

    public static final Fruit APPLE;
    public static final Fruit PEAR;
    public static final Fruit PEACH;
    public static final Fruit ORANGE;
    private static final Fruit $VALUES[];

    static {
        APPLE = new Fruit("APPLE", 0);
        PEAR = new Fruit("PEAR", 1);
        PEACH = new Fruit("PEACH", 2);
        ORANGE = new Fruit("ORANGE", 3);
        $VALUES = (new Fruit[] {
            APPLE, PEAR, PEACH, ORANGE});
    }
}
```

除了对应的四个枚举常量外，还有一个私有的数组，数组中的元素就是枚举常量。编译器自动生成了一个 private 的构造方法，这个构造方法中直接调用父类的构造方法，传入了一个字符串和一个整型变量。从初始化语句中可以看到，字符串的值就是声明枚举常量时使用的名称，而整型变量分别是它们的顺序（从0开始）。枚举类的实现使用了一种多例模式，只有有限的对象可以创建，无法显示调用构造方法创建对象。

values() 方法返回枚举常量数组的一个浅拷贝，可以通过这个数组访问所有的枚举常量；而 valueOf() 则直接调用父类的静态方法 Enum.valueOf()，根据传入的名称字符串获得对应的枚举对象。

## 枚举类的多态

枚举类无法实现继承，但是可以实现接口。

```java
interface WhatName {
  void whatName();
}

public enum Fruit implements whatName {
  Apple {
    @Override
    public void whatName() {
      System.out.println("this is an apple");
    }
  },
  PEAR; // 没有whatname方法
  // common name
  @Override
  public void whatName() {
    System.out.println("this is a fruit");
  }
```

上述代码会扩充编译器生成的源码,以匿名内部类的方式对Fruit进行了Overide:

```java
 APPLE = new Fruit("APPLE", 0);
 // -->
  APPLE = new Fruit("APPLE", 0) {  
    public void whatName()  {  
        System.out.println("this is an apple");  
    }  
   };  
```

