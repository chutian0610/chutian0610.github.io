# Java 依赖注入规范:javax-inject

JSR-330 是Java的依赖注入规范。标准的依赖注入的类在`javax.inject`中定义。

<!--more-->

## `@Inject`

标记可注入的构造函数、方法和字段。

- 可以应用于静态成员和实例成员。
- 可注入成员可以具有任何访问修饰符(private、package-private、protected、public）。
- 首先注入构造函数，然后是字段，最后是方法。
- 超类中的字段和方法在子类中的字段和方法之前被注入。
- 没有指定同一类中的字段和方法之间的注入顺序。

可注入的构造函数用`@Inject`注释，并且接受零个或多个依赖项作为参数。每个类最多只能应用@Inject在一个构造函数上。

```java
@Inject ConstructorModifiers[opt] SimpleTypeName(FormalParameterList[opt]) Throws[opt] ConstructorBody
```

`@Inject`对于公共无参数构造函数是可选的(当前类没有其他构造函数的情况下)。这使得注入器能够调用默认构造函数。

```java
@Inject[opt] Annotations[opt] public SimpleTypeName() Throws[opt] ConstructorBody
```

可注入字段：

- 使用`@Inject注释`。
- 不是final的。
- 可以有任何其他有效的名称。

```java
@Inject FieldModifiers[opt] Type VariableDeclarators;
```

可注入方法：

- 使用`@Inject`注释。
- 不是抽象的。
- 不要声明自己的类型参数。
- 可以返回一个结果。
- 可以有任何其他有效的名称。
- 接受零个或多个依赖项作为参数。

```java
@Inject MethodModifiers[opt] ResultType Identifier(FormalParameterList[opt]) Throws[opt] MethodBody
```

用`@Inject`注解的方法覆盖了用`@Inject`注解的方法，对于每个实例的每个注入请求只注入一次。如果一个方法没有`@Inject注解`，但是覆盖了一个`@Inject`注解的方法，那么这个方法将不会被注入。

需要注入带有@Inject注解的成员。虽然可注入成员可以使用任何可访问性修饰符(包括private)，但平台或注入器限制(如安全性限制或缺少反射支持)可能会阻止非公共成员的注入。

### Qualifiers(限定符)

`@Qualifier`可以注解可注入的字段或参数，并结合类型标识要注入的实现。限定符是可选的，当在独立于注入器的类中与`@Inject`一起使用时，不应该有多个限定符注解单个字段或参数。

```java
public class Car {
    @Inject
    private @Leather Provider<Seat> seatProvider;

    @Inject
    void install(@Tinted Windshield windshield, @Big Trunk trunk) {...}
}
```

如果一个可注入方法覆盖另一个方法，则覆盖方法的参数不会自动从覆盖方法的参数继承限定符。

### Injectable Values(可注入值)

对于给定类型T和可选限定符，注入器必须能够注入用户指定的类:

- 赋值是否与T和相容.
- 具有可注入构造函数。

例如，用户可以使用外部配置来选择T的实现，除此之外，注入的值取决于注入器实现及其配置。

### Circular Dependencies(循环依赖)

检测和解析循环依赖关系是留给注入器实现的练习。两个构造函数之间的循环依赖关系是一个明显的问题，但是您也可以在可注入字段或方法之间有循环依赖关系:

```java
class A {
    @Inject B b;
}
class B {
    @Inject A a;
}
```

构造A的一个实例时,不明智的注入器实现可能会进入一个无限循环构造的实例，B引用A,A的第二个实例引用B,B的第二个实例上引用第二个实例的A,等等。

保守的注入器可能在构建时检测到循环依赖项并生成错误，此时程序员可以通过注入`Provider<A>`或`Provider<B>`来打破循环依赖项，而不是分别注入A或B。直接从注入的构造函数或方法调用提供程序上的get(),会破坏提供程序跳出循环依赖项的能力。在方法或字段注入的情况下，确定依赖项的作用域(例如使用Singleton 作用域)也可以启用有效的循环关系

## `@Qualifier`

限定符标记注解。任何人都可以定义一个新的限定符。一个限定符注释:

- 是用@Qualifier、@Retention(RUNTIME)和@Documentation注解的。
- 可以包含属性
- 可能是公共API的一部分，很像依赖项类型，但不像实现类型不需要是公共API的一部分。
- 如果使用@Target注解，可能会限制使用。虽然该规范只涉及对字段和参数应用限定符，但一些注入器配置可能会在其他地方(例如在方法或类上)使用限定符注解。

例如：

```java
@java.lang.annotation.Documented
@java.lang.annotation.Retention(RUNTIME)
@javax.inject.Qualifier
public @interface Leather {
    Color color() default Color.TAN;
    public enum Color {RED, BLACK, TAN}
}
```

## `@Named`

基于字符串的Qualifier。

```java
@Qualifier
@Documented
@Retention(RUNTIME)
public @interface Named {

    /** The name. */
    String value() default "";
}
```

使用实例：

```java
public class Car {
    @Inject @Named("driver") Seat driverSeat;
    @Inject @Named("passenger") Seat passengerSeat;
    ...
 }
```

## `Provider`

`Provider`提供`T`的实例。通常由注入器实现。对于可以注入的任何类型`T`，您也可以注入`Provider<T>`。与直接注入`T`相比，注入`Provider<T>`可以:
 
- 获取多个实例。
- 延后或者可选地获取一个实例。
- 避免循环依赖。
- 抽象作用域，以便您可以从包含作用域中的实例查找较小作用域中的实例。

例如：

```java
public class Car {
    @Inject public Car(Provider<Seat> seatProvider) {
        Seat driver = seatProvider.get();
        Seat passenger = seatProvider.get();
        ...
    }
}
```

## `@Scope`

Scope标记作用域注解。作用域注解应用于包含可注入构造函数的类，并控制注入器如何重用该类型的实例。默认情况下，如果没有作用域注解，注入器将创建一个实例(通过注入类型的构造函数)，使用该实例进行一次注入，然后忘记它。如果存在作用域注解，则注入器可以保留实例，以便在以后的注入中重用。如果多个线程可以访问一个作用域实例，那么它的实现应该是线程安全的。作用域本身的实现由注入器决定。

接下来的例子中，作用域注解@Singleton确保我们只有一个Log实例：

```java
@Singleton
class Log {
    void log(String message) { ... }
}
``` 

如果注入器在同一类上遇到多个作用域注解或不支持的作用域注解，则会生成错误。

一个作用域注解：
- 是用@Scope、@Retention(RUNTIME)和@Documentation注释的。
- 没有属性。
- 通常不是@inheritance，所以作用域与实现继承是正交的。
- 如果使用@Target注释，可能会限制使用。虽然该规范只涵盖将作用域应用于类，但一些注入器配置可能会在其他地方(例如工厂方法结果)使用作用域注解。

例如：

```java
@java.lang.annotation.Documented
@java.lang.annotation.Retention(RUNTIME)
@javax.inject.Scope
public @interface RequestScoped {}
```

使用`@Scope`注解于作用域注解，可以帮助注入器检测：程序员在类上使用作用域注解但忘记在注入器中配置作用域的情况。保守的注入器将生成错误，而不是不应用于作用域。

## `@Singleton`

标识注入器只实例化一次的类型。不继承。

## 参考

- [1] [JSR-330](http://javax-inject.github.io/javax-inject/)

