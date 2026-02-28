# Java泛型

泛型在定义类，接口和方法时使类型（类和接口）成为参数。与方法声明中使用的更熟悉的形式参数非常相似，类型参数为您提供了一种使用不同输入重复使用相同代码的方法。区别在于形式参数的输入是值，而类型参数的输入是类型。使用泛型带来了下面几点优势：

* 消除代码中的直接转型。

    ```java
    List<String> list = new ArrayList<String>();
    list.add("hello");
    String s = list.get(0);   // no cast
    ```

* Java编译器对泛型代码进行强类型检查，如果代码违反类型安全，则会发出错误。修复编译时错误比修复运行时错误容易，后者可能很难找到。
* 通过使用泛型，可以实现对不同类型的集合工作的泛型算法，可以对其进行自定义，类型安全且易于阅读。

<!--more-->

## 泛型类

首先，我们用包装类来举个例子：

```java
public class Wrapper {
    private Object object;

    public void set(Object object) { this.object = object; }
    public Object get() { return object; }
}
```

由于其方法接受或返回Object，因此只要它不是原始类型之一，就可以随意传递任何所需的内容。在编译时无法验证类的使用方式。代码的一部分可能会将Integer放在框中，并期望从中取出Integer，而代码的另一部分可能会错误地传入String，从而导致运行时错误。

下面是Wrapper的泛型版本：

```java
public class Wrapper<T> {
    // T 是泛型参数
    private T t;

    public void set(T t) { this.t = t; }
    public T get() { return t; }
}

// 要从代码中引用泛型Wrapper类，必须执行泛型调用，该调用将T替换为某些具体值，
// 例如Integer.此时的Integer 是类型变量，代表实际的Integer Class。
Wrapper<Integer> integerWrapper = new Wrapper<>();
```

所有出现的Object都由T代替。类型变量可以是您指定的任何非原始类型：任何类类型，任何接口类型，任何数组类型，甚至另一个类型变量。

> * 类型参数命名约定: 类型参数名称是单个大写字母。例如，E,K,T
> * 在Java SE 7和更高版本中，只要编译器可以从上下文确定或推断出类型参数，就可以用一组空的类型参数`<>`替换调用通用类的构造函数所需的类型参数。正如上面的`new Wrapper<>()`。

泛型类可以具有多个类型参数。

```java
public interface Pair<K, V> {
    public K getKey();
    public V getValue();
}

public class OrderedPair<K, V> implements Pair<K, V> {

    private K key;
    private V value;

    public OrderedPair(K key, V value) {
      this.key = key;
      this.value = value;
    }

    public K getKey(){ return key; }
    public V getValue() { return value; }
}

Pair<String, Integer> p1 = new OrderedPair<>("Even", 8); // 注意参数类型的变化
Pair<String, String>  p2 = new OrderedPair<>("hello", "world")
```

还可以用参数化类型(例如`List<String>`)替换类型参数（K或V）

```java
OrderedPair<String, Wrapper<Integer>> p = new OrderedPair<>("primes", new Wrapper<Integer>(...));
```

### 原始类型

以Wrapper为例，原始类型是初始化时不提供类型参数： `Wrapper rawWrapper = new Wrapper();`原始类型默认会有Object作为类型参数。

```java
Wrapper<String> stringWrapper = new Wrapper<>();
Wrapper rawWrapper = stringWrapper;               // OK

Wrapper rawWrapper = new Wrapper();           // rawBox is a raw type of Box<T>
Wrapper<Integer> intWrapper = rawWrapper;     // warning: unchecked conversion

Wrapper<String> stringWrapper = new Wrapper<>();
Wrapper rawWrapper = stringWrapper;
rawWrapper.set(8);  // warning: unchecked invocation to set(T)
```

原始类型会绕过通用类型检查，从而将不安全代码的捕获推迟到运行时。因此，应避免使用原始类型。使用JVM 选项`-Xlint:unchecked`，开启泛型类型检查，默认是关闭的。`-Xlint:-unchecked` 选项可以关闭泛型类型检查。

## 泛型方法

泛型方法是引入自己的类型参数的方法。这类似于泛型类，但是类型参数的范围仅限于声明它的方法。泛型方法包括静态和非静态泛型方法，以及泛型类构造函数。

泛型方法的语法包括类型参数列表(在尖括号内)，类型参数部分必须出现在方法的返回类型之前。

```java
public class Util {
  //  泛型方法
  public static <K, V> boolean compare(Pair<K, V> p1, Pair<K, V> p2) {
    return p1.getKey().equals(p2.getKey()) && 
      p1.getValue().equals(p2.getValue());
  }
}
public class Pair<K, V> {

  private K key;
  private V value;

  public Pair(K key, V value) {
    this.key = key;
    this.value = value;
  }

  public void setKey(K key) { this.key = key; }
  public void setValue(V value) { this.value = value; }
  public K getKey()   { return key; }
  public V getValue() { return value; }
}
```

## 有界类型参数

有界类型参数可以限制在参数化类型中用作类型参数的类型，例如，对数字进行操作的方法可能只希望接受Number或其子类的实例。要声明有界的类型参数，请列出类型参数的名称，后跟extends关键字，然后是其上限（在本示例中为）Number。请注意，在这种情况下，extends一般意义上是指“扩展”（如在类中）或“实现”（如在接口中）。

```java
public class Box<T> {

    private T t;

    public void set(T t) {
        this.t = t;
    }

    public T get() {
        return t;
    }

    public <U extends Number> void inspect(U u){
        System.out.println("T: " + t.getClass().getName());
        System.out.println("U: " + u.getClass().getName());
    }

    public static void main(String[] args) {
        Box<Integer> integerBox = new Box<Integer>();
        integerBox.set(new Integer(10));
        integerBox.inspect("some text"); // 不满足 Number 的约束
    }
}
```

### 交集类型参数

一个类型参数可以具有多个界限：

```java
<T extends B1 & B2 & B3>
```

具有多个界限的类型变量是界限中列出的所有类型的子类型。如果范围之一是类，则必须首先指定它。例如：

* 类 A
* 接口 B
* 接口 C

```java
class D <T extends A & B & C>
// 如果未首先指定绑定A，则会出现编译时错误：
```

## 继承

众所周知，只要类型兼容，就可以将一种类型的对象分配给另一种类型的对象。例如，你可以指定一个整数一个对象，因为对象是一个整数的超类型.

```java
public void boxTest(Box<Number> n)
```

对于上面的方法，能否传递`Box<Integer>`或`Box<Double>`作为参数？答案是不能的，因为`Box<Integer>`和`Box<Double>`不是`Box<Number>`的子类型。

给定两个具体类型A和B（例如Number和Integer），无论A和B是否相关，`MyClass <A>`与`MyClass <B>`没有关系。`MyClass <A>`和`MyClass <B>`的公共父对象是Object。

可以通过扩展或实现来泛化通用类或接口。一个类或接口的类型参数与另一类或接口的类型参数之间的关系由extend和Implements子句确定。以Collections类为例，`ArrayList<E>`实现`List <E>`，而`List<E>`扩展`Collection<E>`。因此`ArrayList<String>`是`List<String>`的子类型，而`List<String>`是`Collection<String>`的子类型。只要您不改变类型参数，子类型关系就保留在类型之间。

当然，我们也可以使用扩展。

```java
interface PayloadList<E,P> extends List<E> {
  void setPayload(int index, P val);
}
```

## 类型推断

类型推断是Java编译器查看每个方法调用和相应声明以确定使调用适用的类型参数的能力。推理算法确定参数的类型，以及确定结果是否已分配或返回的类型（如果有）。最后，推理算法尝试找到与所有参数一起使用的最具体的类型。

```java
// 传递给pick方法的两个参数的类型为Serializable
static <T> T pick(T a1, T a2) { return a2; }
Serializable s = pick("d", new ArrayList<String>());
```

### 类型推断和泛型方法

泛型方法引入了类型推断功能。

```java
public static <U> void addBox(U u,java.util.List <Box<U>> box){
    Box <U> box = new Box <>();
    box.set(u);
    box.add(box);
}
```

方法定义了一个类型参数U,java 编译器可以推断泛型的方法的类型参数。

```java
BoxDemo.<Integer>addBox(Integer.valueOf(10), listOfIntegerBoxes);
// 由于java 编译器可以推断出泛型，可以省去 <Integer>
BoxDemo.addBox(Integer.valueOf(20), listOfIntegerBoxes);
```

### 类型推断和泛型类实例化

可以用空的类型参数`<>`替换类的构造函数所需的类型参数，只要编译器可以从上下文中推断出类型参数即可。

例如:

```java
Map <String，List <String >> myMap = new HashMap <String，List <String >>();
```

可以被改写成:

```java
Map <String，List <String >> myMap = new HashMap <>();
```

### 类型推断和构造器

```java
class MyClass<X> {
  <T> MyClass(T t) {
    // ...
  }
}
```

对于上面的class,当我们使用`new MyClass<Integer>("")`来初始化时,会默认创建类型为`MyClass<Integer>`的实例，Integer被默认识别成类型参数X。而构造器的参数T被识别成String(基于参数类型)。

### 目标类

Java编译器可以使用目标类型来推断通用方法调用的类型参数。以`Collections.emptyList`为例:

```java
// 方法申明
static <T> List<T> emptyList();
// 该语句期望一个实例List<String>；此数据类型是目标类型。因为方法emptyList返回类型List<T>，所以编译器推断类型参数T必须是String
List<String> listOne = Collections.emptyList();
```

在java7中，目标推断还不能很好地处理方法参数:

```java
void processStringList(List<String> stringList) {
    // process stringList
}
processStringList(Collections.emptyList());
// 编译报错:List<Object> cannot be converted to List<String>
```

但是在Java 8中，这个问题得到了修复。

## 类型参数通配符

在泛型中，一个常规的通配符是`?`,表示任意未知类型。通配符可以在多种情况下使用：作为参数，字段或局部变量的类型；有时作为返回类型。通配符从不用作泛型方法调用，泛型类实例构造或超类的类型参数。

### 上界通配符

可以使用上限通配符来放宽对变量的限制。例如，当想使用一个方法操作`List<Integer>,List<Double>, List<Number>`时，可以使用上界通配符来处理。要声明上界通配符，请使用通配符`?`，后跟extend关键字，然后是其上界。在这种情况下，extends通常用于表示“扩展”（如在类中）或“实现”（如在接口中）。`List<Number>` 比`List<? extends Number>`有更多限制。因为前者仅匹配Number类型的列表，而后者匹配Number类型或其任何子类的列表。

```java
public static void process(List<? extends Foo> list) {
    for (Foo elem : list) {
        // ...
    }
}
```

### 无限通配符

使用通配符`?`指定无界通配符类型，例如`List<?>`。这称为未知类型列表。在两种情况下，无界通配符是一种有用的方法：

* 编写一个可以使用Object类中提供的功能实现的方法。
* 当代码使用通用类中不依赖于type参数的方法时。例如，List.size或List.clear。

### 下界通配符

上界通配符将未知类型限制为特定类型或该类型的子类型，并使用extend关键字表示。以类似的方式，下界通配符将未知类型限制为特定类型或该类型的超类型。

下限通配符使用通配符`?`表示，后跟super关键字，后跟下限：`<？super A>`。

> 可以为通配符指定一个上限，也可以指定一个下限，但不能同时指定两者。

### 通配符和子类型

给定Integer是Number的子类型，`List <Integer>`和`List <Number>`之间是什么关系？

尽管Integer是Number的子类型，但`List <Integer>`不是`List <Number>`的子类型，实际上，这两种类型无关。`List <Number>`和`List <Integer>`的公共父级是`List <？>`。

![](generics-wildcardSubtyping.gif)

### 通配符使用准则

将变量视为提供以下两个功能之一:

* “输入”变量将数据提供给代码。
* “输出”变量保存要在其他地方使用的数据。

> 当然，某些变量既用于“输入”又用于“输出”目的。

通配符准则： 

* 使用extends关键字，使用上限通配符定义“输入”变量。
* 使用super关键字使用下界通配符定义“输出”变量。
* 如果可以使用Object类中定义的方法访问“输入”变量，请使用无界通配符。
* 如果代码需要同时使用“输入”和“输出”变量来访问变量，则不要使用通配符。

## 类型擦除

Java语言引入了泛型，以在编译时提供更严格的类型检查并支持泛型编程。为了实现泛型，Java编译器将类型擦除应用于：

* 如果类型参数不受限制，则将通用类型中的所有类型参数替换为其边界或对象。因此，产生的字节码仅包含普通的类，接口和方法。
* 必要时插入类型转换，以保持类型安全。
* 生成桥接方法以在扩展的泛型类型中保留多态。

> 类型擦除可确保不会为参数化类型创建新的类；因此，泛型不会产生运行时开销。

### 桥接方法

```java
public class Node<T> {
    public T data;

    public Node(T data) { this.data = data; }
    public void setData(T data) {
        System.out.println("Node.setData");
        this.data = data;
    }
}
public class MyNode extends Node<Integer> {
    public MyNode(Integer data) { super(data); }
    public void setData(Integer data) {
        System.out.println("MyNode.setData");
        super.setData(data);
    }
}
```

类型擦除后，Node和MyNode类变为：

```java
public class Node {

    public Object data;

    public Node(Object data) { this.data = data; }

    public void setData(Object data) {
        System.out.println("Node.setData");
        this.data = data;
    }
}

public class MyNode extends Node {

    public MyNode(Integer data) { super(data); }

    public void setData(Integer data) {
        System.out.println("MyNode.setData");
        super.setData(data);
    }
}
```

类型擦除后，方法签名不匹配。因此，MyNode setData方法不会覆盖Node setData方法。

为了解决此问题并在类型擦除后保留通用类型的 多态性，Java编译器生成了一个桥接方法，以确保子类型能够按预期工作。对于MyNode类，编译器为setData生成以下桥接方法：

```java
class MyNode extends Node {

    // Bridge method generated by the compiler
    //
    public void setData(Object data) {
        setData((Integer) data);
    }

    public void setData(Integer data) {
        System.out.println("MyNode.setData");
        super.setData(data);
    }

    // ...
}
```

## 如何优化类型擦除

由于类型擦除，在运行时无法得知泛型的类型。所以我们需要通过反射保持泛型信息。

```java
public abstract class TypeInfo<T>
{
    protected final Type type;

    public Type getType() {
        return type;
    }
    protected TypeInfo(){
        Type superClass = getClass().getGenericSuperclass();

        Type type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
        this.type = type;
    }

    // 注意，使用时需要使用匿名内部类方式
    public final static Type LIST_STRING = new TypeInfo<List<String>>() {}.getType();
}
```

## 参考

- [1] [Oracle Java Tutorial.generics](https://docs.oracle.com/javase/tutorial/java/generics/)

