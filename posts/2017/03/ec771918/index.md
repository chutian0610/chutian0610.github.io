# Java Lambda Function

如果匿名类的实现非常简单，比如只包含一个方法的接口，那么匿名类的语法可能看起来很笨重且不明确。使用Lambda表达式可以更简洁地表达单方法类的实例。

<!--more-->

看一个[来自oracle tutorial 文档](https://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html)的例子：

```java
interface CheckPerson {
    boolean test(Person p);
}

// 声明一个实现类
class CheckPersonEligibleForSelectiveService implements CheckPerson {
    public boolean test(Person p) {
        return p.gender == Person.Sex.MALE &&
            p.getAge() >= 18 &&
            p.getAge() <= 25;
    }
}

public static void printPersons(
    List<Person> roster, CheckPerson tester) {
    for (Person p : roster) {
        if (tester.test(p)) {
            p.printPerson(); // 打印个人信息
        }
    }
}

printPersons(roster, new CheckPersonEligibleForSelectiveService()); // 输出符合条件的人的个人信息

// 匿名内部类
printPersons(
    roster,
    new CheckPerson() {
        public boolean test(Person p) {
            return p.getGender() == Person.Sex.MALE
                && p.getAge() >= 18
                && p.getAge() <= 25;
        }
    }
);
```

显然为了实现过滤元素，而声明一个实现类，亦或是实现一个匿名类，在此处都显得过于重了。我们可以使用lambda表达式来优化代码，提高可读性。

```java
@FunctionalInterface
// 此处将接口定义函数式接口是必须的
interface CheckPerson {
    boolean test(Person p);
}

printPersons(
    roster,
    (Person p) -> p.getGender() == Person.Sex.MALE
        && p.getAge() >= 18
        && p.getAge() <= 25
);
```

## 使用`@FunctionalInterface`自定义lambda表达式

`@FunctionalInterface`该注解用于标志该接口为Java函数式接口。一个函数式接口只能有一个抽象方法。默认方法`default method`拥有自己的实现，并不是抽象的。

如果接口定义的一个抽象方法覆盖了`java.lang.Object`的一个public方法，那么这个抽象方法不被计入。

## 使用标准的Lambda函数接口

JDK在`java.util.function`包中定义了一些通用接口:

### `Consumer<T>`

代表了接收一个入参，无返回值的操作。Consumer可以通过副作用来操作对象。

```java
@FunctionalInterface
public interface Consumer<T> {

    void accept(T t);
    // 返回一个混合的Consumer,当前操作完成后会执行after操作
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

### `BiConsumer<T,U>`

代表了接受两个入参，无返回值的操作。是Consumer的二元特例。

```java
@FunctionalInterface
public interface BiConsumer<T, U> {
    void accept(T t, U u);
    ...
}
```

### `Function<T,R>`

代表了接收一个入参，返回一个结果的函数。

```java
@FunctionalInterface
public interface Function<T, R> {

    R apply(T t);

    // 返回混合方法，首先执行before
    default <V> Function<V, R> compose(Function< super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }
    // 返回混合方法，后执行after
    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }
    // 返回一个始终返回入参的函数
    static <T> Function<T, T> identity() {
        return t -> t;
    }
}
```

### `BiFunction<T, U, R>`

代表了接收两个入参，返回一个结果的函数，这是Function的二元特例。

```java
@FunctionalInterface
public interface BiFunction<T, U, R> {
    R apply(T t, U u);
    ...
}
```

### `BinaryOperator<T>`

代表了两个入参和结果都是同一种类型的操作。这是BiFunction的一种特例。

一个比较常见的例子是，去两个对象中的较小值或较大值。

```java
@FunctionalInterface
public interface BinaryOperator<T> extends BiFunction<T,T,T> {
    public static <T> BinaryOperator<T> minBy(Comparator<? super T> comparator) {
        Objects.requireNonNull(comparator);
        return (a, b) -> comparator.compare(a, b) <= 0 ? a : b;
    }
    public static <T> BinaryOperator<T> maxBy(Comparator<? super T> comparator) {
        Objects.requireNonNull(comparator);
        return (a, b) -> comparator.compare(a, b) >= 0 ? a : b;
    }
}
```

### `Predicate<T>`

代表了一个断言方法，该方法接收一个参数，返回布尔值。

```java
@FunctionalInterface
public interface Predicate<T> {

    boolean test(T t);
    // 混合断言方法
    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }
    // 断言-非 方法
    default Predicate<T> negate() {
        return (t) -> !test(t);
    }
    // 断言-或 方法
    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }
    // 断言相等
    static <T> Predicate<T> isEqual(Object targetRef) {
        return (null == targetRef)
                ? Objects::isNull
                : object -> targetRef.equals(object);
    }
}
```

### `BiPredicate<T, U>`

代表了一个断言方法，该方法接收两个参数，返回布尔值。是Predicate的二元实现。

### `Supplier<T>`

代表结果的提供者，无入参。

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

## lambda 的类型

如何确定Lambda表达式的类型？以选择年龄在18至25岁之间的男性成员的lambda表达式为例

```java
p-> p.getGender（）== Person.Sex.MALE 
    && p.getAge（）> = 18 
    && p.getAge（）<= 25
```

* 当Java调用方法printPersonsWithPredicate时，它期望的数据类型为`Predicate<Person>`.
* 当Java调用方法printPersons时，它期望的数据类型为CheckPerson。

```java 
public void printPersonsWithPredicate(List<Person> roster, Predicate<Person> tester)
public static void printPersons(List<Person> roster, CheckPerson tester)
```

上述通过方法判断的期望数据类型被称为目标类型。为了确定lambda表达式的类型，Java编译器使用找到lambda表达式上下文的目标类型。因此，您只能在Java编译器可以确定目标类型的情况下使用lambda表达式：

* 变量声明,Variable declarations
* 赋值,Assignments
* 返回值,Return statements
* 数组初始化,Array initializers
* 方法参数,Method or constructor arguments
* lambda body体:Lambda expression bodies
* 条件表达式,Conditional expressions,`?:`
* 转型表达式,Cast expressions

