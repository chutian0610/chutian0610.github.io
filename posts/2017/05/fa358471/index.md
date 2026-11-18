# Java 反射

反射常被用于在程序运行时检查或修改其执行行为。

<!--more-->

## 为什么要使用反射

反射可以在下面几点帮助我们

* 扩展功能
    - 扩展已有类的功能，例如有些字段不支持访问或修改，可以使用反射扩展。
* 类浏览器和可视化开发
    - 类浏览器需要枚举类的成员，广泛应用于可视化开发环境
* 调试和测试工具
    - 调试器需要能够检查类的私有成员，测试工具会使用反射调用API。

## 反射的弊端

反射很强大，但是不要滥用，如果可以避免使用，尽量避免。当使用反射时，下面的问题需要注意。

* 性能开销
    - 反射会动态地处理对象，导致JVM的优化不能进行，因此，要避免在性能敏感地地方使用反射。
* 安全限制
    - 反射可能会与安全管理相互冲突。
* 内部资源暴露
    - 反射可以访问private方法，这可能会带来意料之外的副作用。

## 反射与类

java中的对象不是基本值类型就是引用类型，而所有的引用类型都继承自`java.lang.Object`。

Java虚拟机为每个类生成一个不可变的`java.lang.Class`的类实例，该类实例提供了对象属性的运行期检查，包括了对象的成员和类型信息。`java.lang.Class`是所有反射操作的入口。

### 如何获取类实例

`java.lang.Class`是所有反射操作的入口。需要调用`java.lang.Class`的对应方法，以获取类实例。

#### `Object.getClass()`

如果可以获取到一个对象的实例，最简单的获取类实例的方法是调用该对象的`getClass()`方法。注意，这只是用于引用类型\(`getClass()`方法继承自`Object`\)。下面是几个例子:

* 获取`String`的类实例

```java
Class c = "foo".getClass();
```

* 获取`Enum`的类实例

```java
enum E{A,B}
Class c= A.getClass();
```

* 获取集合的类实例

```java
Set<String> s = new HashSet<String>();
Class c = s.getClass();
```

* 获取数组的类实例

```java
byte[] bytes = new byte[1024];
Class c = bytes.getClass();
```

#### `.class` 语法

如果我们可以获得类型但是无法获取对应的对象，那么可以使用`.class`语法来获得类实例。这同时解决了基本值类型的类实例获取方法。

* 获取基本值类型的类实例

```java
boolean b;
Class c = boolean.class;  // correct
```

* 获取引用类型的类实例

```java
Class c = java.io.PrintStream.class;
```

* 获取数组类型的类实例

```java
Class c = int[][][].class;
```

#### `Class.forName()`

如果可以获得一个类的fully-qualified name\(包名+类名\)，那么可以通过static方法`Class.forName()`来获得对应的类实例。注意，基本值类型不能使用这个方法。

* String

```java
Class c = Class.forName("java.lang.String");
```

#### 数组类型的`Class.forName()`

* double一维数组

```java
Class cDoubleArray = Class.forName("[D");
```

> 对应`double[].class.getName()`

* String 二维数组

```java
Class cStringArray = Class.forName("[[Ljava.lang.String;");
```

> 对应`String[][].class.getName()`

#### 基本值类型通过包装类获取类实例

每个基本值类型和void都有自己对应的包装类\(java.lang包中\)，因此可以通过包装类的`TYPE`属性来获取对应的封装类的类实例。

```java
Class c = Double.TYPE;
Class c = Void.TYPE;
```

### `Class`的其它方法

#### `Class.getSuperclass()`

返回当前类实例的父类。

#### `Class.getClasses()`

返回`Class<?>[]`，包含了当前类实例的所有public成员类，接口，枚举\(包括继承的成员\)的类实例。

```java
Class<?>[] c = Character.class.getDeclaredClasses();

/**
* 数组c中会包含Character类中的3个public成员类
* static class    Character.Subset
* static class    Character.UnicodeBlock
* static class    Character.UnicodeScript
*/
```

#### `Class.getDeclaredClasses()`

返回`Class<?>[]`，包含了当前类实例的所有\(public, protected, friendlly, private \)成员类，接口，枚举\(包括继承的成员\)的类实例。

```java
Class<?>[] c = Character.class.getDeclaredClasses();

/**
* 数组c中会包含Character类中的3个public成员类和一个私有类
* Character.Subset
* Character.UnicodeBlock
* Character.UnicodeScript
* (private)Character.CharacterCache.
*/
```

#### `getDeclaringClass()`

* `java.lang.Class.getDeclaringClass()`
* `java.lang.reflect.Field.getDeclaringClass()`
* `java.lang.reflect.Method.getDeclaringClass()`
* `java.lang.reflect.Constructor.getDeclaringClass()`

获取Class，Field，Method，Constructor声明所在类，注意，匿名内部类没有声明所在类。

```java
Field f = System.class.getField("out");
Class c = f.getDeclaringClass();
//  c -> System.

public class MyClass {
    static Object o1 = new Object() {
        public void m() {} 
    };
    static Class o = o1.getClass().getEnclosingClass();
}
//  o -> null.
```

#### `Class.getEnclosingClass()`

返回匿名内部类的封装类。

```java
public class MyClass {
    static Object o1 = new Object() { 
        public void m() {} 
    };
    static Class o = o1.getClass().getEnclosingClass();
    // o -> MyClass
}
```

### 类的标识符和类型检查

一个类有一个或多个修饰符，这些修饰符会影响运行期行为。

* 访问权限修饰符
    - public
    - protected
    - friendly\(默认包权限，也被称作default\)
    - private
* 重载修饰符
    - abstract
* 静态修饰符
    - static
* 不可修改修饰符
    - final
* 强制严格浮点操作修饰符
    - strictfp
* 注解

`java.lang.reflect.Modifer`包含了所有可能的修饰符的声明，也包含了解析`Class.getModifiers()`返回值的方法。

#### 获取修饰符的例子

```java
import java.lang.annotation.Annotation;
import java.lang.reflect.Modifier;
import java.lang.reflect.Type;
import java.lang.reflect.TypeVariable;
import java.util.ArrayList;
import java.util.List;
import static java.lang.System.out;

/**
 * 引用自 
 * http://docs.oracle.com/javase/tutorial/reflect/class/classModifiers.html
 */
public class ClassDeclarator {
    public static void main(String... args) {
        try {
            Class<?> c = Class.forName(args[0]);
            out.format("Class:%n  %s%n%n", c.getCanonicalName());
            out.format("Modifiers:%n  %s%n%n",Modifier.toString(c.getModifiers()));
            out.format("Type Parameters:%n");
            TypeVariable[] tv = c.getTypeParameters();
            if (tv.length != 0) {
                out.format("  ");
                for (TypeVariable t : tv)
                    out.format("%s ", t.getName());
                out.format("%n%n");
            } else {
                out.format("  -- No Type Parameters --%n%n");
            }
            out.format("Implemented Interfaces:%n");
            Type[] intfs = c.getGenericInterfaces();
            if (intfs.length != 0) {
                for (Type intf : intfs)
                    out.format("  %s%n", intf.toString());
                out.format("%n");
            } else {
                out.format("  -- No Implemented Interfaces --%n%n");
            }
            out.format("Inheritance Path:%n");
            List<Class> l = new ArrayList<Class>();
            printAncestor(c, l);
            if (l.size() != 0) {
                for (Class<?> cl : l)
                    out.format("  %s%n", cl.getCanonicalName());
                out.format("%n");
            } else {
                out.format("  -- No Super Classes --%n%n");
            }
            out.format("Annotations:%n");
            Annotation[] ann = c.getAnnotations();
            if (ann.length != 0) {
                for (Annotation a : ann)
                    out.format("  %s%n", a.toString());
                out.format("%n");
            } else {
                out.format("  -- No Annotations --%n%n");
            }
        } catch (ClassNotFoundException x) {
            x.printStackTrace();
        }
    }
    private static void printAncestor(Class<?> c, List<Class> l) {
        Class<?> ancestor = c.getSuperclass();
        if (ancestor != null) {
            l.add(ancestor);
            printAncestor(ancestor, l);
        }
    }
}
```

### 探索类的成员

`java.lang.Class`中获取fields,methods和constructors的方法分为两种，一种是获取全部成员，将结果放入List；一种是获取名称对应的成员。
另外，根据成员的声明位置，获取成员的方法也分为两种，一种是直接获取类中所有声明的成员，一种是获取类的public成员\(包括继承的成员\)。

JAVA 使用 `java.lang.reflect.Member`接口代表反射使用的类成员。Member成员包含以下几种:

* `java.lang.reflect.Field`
* `java.lang.reflect.Method`
* `java.lang.reflect.Constructor`

#### `field`

|Class API| List of members ?| Inherited members ?| Private members ?|
|:---|:---|:---|:---|
|`getDeclaredField()`|no|no|yes|
|`getField()`|no|yes|no|
|`getDeclaredFields()`|yes|no|yes|
|`getFields()`|yes|yes|no|


> 注意: getDeclaredFields &  getFields 返回顺序不稳定

#### `method`

|Class API| List of members ?| Inherited members ?| Private members ?|
|:---|:---|:---|:---|
|`getDeclaredMethod()`|no|no|yes|
|`getMethod()`|no|yes|no|
|`getDeclaredMethods()`|yes|no|yes|
|`getMethods()`|yes|yes|no|

> 注意: getDeclaredMethods & getMethods 返回顺序不稳定

#### `constructor`

|Class API| List of members ?| Private members ?|
|:---|:---|:---|
|`getDeclaredConstructor()`|no|yes|
|`getConstructor()`|no|no|
|`getDeclaredContructors()`|yes|yes|
|`getContructors()`|yes|no|

> 注意: getDeclaredContructors & getContructors 返回顺序不稳定

#### 获取类的成员的代码示例

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Member;
import static java.lang.System.out;

/**
 * 引用自
 * http://docs.oracle.com/javase/tutorial/reflect/class/classMembers.html
 */
enum ClassMember { CONSTRUCTOR, FIELD, METHOD, CLASS, ALL }

public class ClassSpy {
    public static void main(String... args) {
    try {
        Class<?> c = Class.forName(args[0]);
        out.format("Class:%n  %s%n%n", c.getCanonicalName());

        Package p = c.getPackage();
        out.format("Package:%n  %s%n%n",
               (p != null ? p.getName() : "-- No Package --"));

        for (int i = 1; i < args.length; i++) {
        switch (ClassMember.valueOf(args[i])) {
        case CONSTRUCTOR:
            printMembers(c.getConstructors(), "Constructor");
            break;
        case FIELD:
            printMembers(c.getFields(), "Fields");
            break;
        case METHOD:
            printMembers(c.getMethods(), "Methods");
            break;
        case CLASS:
            printClasses(c);
            break;
        case ALL:
            printMembers(c.getConstructors(), "Constuctors");
            printMembers(c.getFields(), "Fields");
            printMembers(c.getMethods(), "Methods");
            printClasses(c);
            break;
        default:
            assert false;
        }
        }
    } catch (ClassNotFoundException x) {
        x.printStackTrace();
    }
    }

    private static void printMembers(Member[] mbrs, String s) {
    out.format("%s:%n", s);
    for (Member mbr : mbrs) {
        if (mbr instanceof Field)
        out.format("  %s%n", ((Field)mbr).toGenericString());
        else if (mbr instanceof Constructor)
        out.format("  %s%n", ((Constructor)mbr).toGenericString());
        else if (mbr instanceof Method)
        out.format("  %s%n", ((Method)mbr).toGenericString());
    }
    if (mbrs.length == 0)
        out.format("  -- No %s --%n", s);
    out.format("%n");
    }

    private static void printClasses(Class<?> c) {
    out.format("Classes:%n");
    Class<?>[] clss = c.getClasses();
    for (Class<?> cls : clss)
        out.format("  %s%n", cls.getCanonicalName());
    if (clss.length == 0)
        out.format("  -- No member interfaces, classes, or enums --%n");
    out.format("%n");
    }
}
```

## 反射中的类成员--Field

`java.lang.reflect.Filed` 提供了对特定对象Field的操作，包括获取类型信息，get和set值。

### 获取Field的类型

* `Field.getType()`

返回field声明类型的类实例。

* `Field.getGenericType()`

返回field对应的泛型类型。

```java
import java.lang.reflect.Field;
import java.util.List;
/**
 * 引用自
 * http://docs.oracle.com/javase/tutorial/reflect/member/fieldTypes.html
 */
public class FieldSpy<T> {
    public boolean[][] b = {{ false, false }, { true, true } };
    public String name  = "Alice";
    public List<Integer> list;
    public T val;

    public static void main(String... args) {
    try {
        Class<?> c = Class.forName(args[0]);
        Field f = c.getField(args[1]);
        System.out.format("Type: %s%n", f.getType());
        System.out.format("GenericType: %s%n", f.getGenericType());
    } catch (ClassNotFoundException x) {
        x.printStackTrace();
    } catch (NoSuchFieldException x) {
        x.printStackTrace();
    }
    }
}
```

### 获取Field的修饰符

field的声明中会包含部分修饰符:

* 权限访问修饰符
  * `public`
  * `protectded`
  * `private`
* Field专属运行期修饰符
  * 该字段不加入序列化`transient`
  * 并发写可见 `volatile`
* 静态修饰符 `static`
* 值不可变修饰符 `final`
* 注解

#### `Field.getModifiers()`

方法返回不同修饰符的Int值，`Modifier`类可以用于解析int值。


```java
import java.lang.reflect.Field;
import java.lang.reflect.Modifier;
import static java.lang.System.out;

enum Spy { BLACK , WHITE }

/**
 * 查询修饰符对应field
 * $ java FieldModifierSpy FieldModifierSpy volatile
 * Fields in Class 'FieldModifierSpy' containing modifiers:  volatile
 * share    [ synthetic=false enum_constant=false ]
 * 引用自
 * http://docs.oracle.com/javase/tutorial/reflect/member/fieldModifiers.html
 */
public class FieldModifierSpy {
    volatile int share;
    int instance;
    class Inner {}

    public static void main(String... args) {
        try {
            Class<?> c = Class.forName(args[0]);
            int searchMods = 0x0;
            for (int i = 1; i < args.length; i++) {
                  searchMods |= modifierFromString(args[i]);
            }

            Field[] flds = c.getDeclaredFields();
            out.format("Fields in Class '%s' containing modifiers:  %s%n",
                   c.getName(),
                   Modifier.toString(searchMods));
            boolean found = false;
            for (Field f : flds) {
                int foundMods = f.getModifiers();
                if ((foundMods & searchMods) == searchMods) {
                    out.format("%-8s [ synthetic=%-5b enum_constant=%-5b ]%n",
                           f.getName(), f.isSynthetic(),
                           f.isEnumConstant());
                    found = true;
                }
            }

            if (!found) {
                  out.format("No matching fields%n");
            }

        } catch (ClassNotFoundException x) {
            x.printStackTrace();
        }
    }

    private static int modifierFromString(String s) {
        int m = 0x0;
        if ("public".equals(s))           m |= Modifier.PUBLIC;
        else if ("protected".equals(s))   m |= Modifier.PROTECTED;
        else if ("private".equals(s))     m |= Modifier.PRIVATE;
        else if ("static".equals(s))      m |= Modifier.STATIC;
        else if ("final".equals(s))       m |= Modifier.FINAL;
        else if ("transient".equals(s))   m |= Modifier.TRANSIENT;
        else if ("volatile".equals(s))    m |= Modifier.VOLATILE;
        return m;
    }
}
```

### `Field.isSynthetic()`

`isSynthetic()`方法可以用来判断field是否是编译器生成的用于内部使用的属性.例如，内部类\(不包括静态内部类\)中会使用字段`this$0`指向外部类的对象引用;enums会使用字段`$VALUES`来实现static方法`values()`。这些`synthetic field`被包含在`Class.getDeclaredFields()`的返回结果中，但不会被包含在`Class.getField()`中，因为这些`synthetic field`通常来说不是public的。

#### 内部类`this$0`

```java
public class OutClass{
  public class InnerClass{

  }
}
```

执行`javac`,`javap -verbose`命令

* `javac`

生成`OutClass.class`,`OutClass$InnerClass.class`文件

* `javap -verbose`

```java
{
  final OutClass this$0;  //此处是对外部类对象的引用
    descriptor: LOutClass;
    flags: ACC_FINAL, ACC_SYNTHETIC // isSynthetic

  public OutClass$InnerClass(OutClass);
    descriptor: (LOutClass;)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: putfield      #1                  // Field this$0:LOutClass;
         5: aload_0
         6: invokespecial #2                  // Method java/lang/Object."<init>":()V
         9: return
      LineNumberTable:
        line 2: 0
}
```

### Field值的get和set

对于某个给定的对象，可以使用反射来给对象的属性赋值，这通常发生在无法通过正常方法赋值的场景下。注意，通过反射赋值违反了类的设计初衷，所以使用是应当尽可能的慎重。


```java
import java.lang.reflect.Field;
import java.util.Arrays;
import static java.lang.System.out;

enum Tweedle { DEE, DUM }

/**
 * field -> set value
 * 引用自
 * http://docs.oracle.com/javase/tutorial/reflect/member/fieldValues.html
 * $ java Book
 * BEFORE:  chapters     = 0
 *  AFTER:  chapters     = 12
 * BEFORE:  characters   = [Alice, White Rabbit]
 *  AFTER:  characters   = [Queen, King]
 * BEFORE:  twin         = DEE
 *  AFTER:  twin         = DUM
 */

public class Book {
    public long chapters = 0;
    public String[] characters = { "Alice", "White Rabbit" };
    public Tweedle twin = Tweedle.DEE;

    public static void main(String... args) {
    Book book = new Book();
    String fmt = "%6S:  %-12s = %s%n";

    try {
        Class<?> c = book.getClass();

        Field chap = c.getDeclaredField("chapters");
        out.format(fmt, "before", "chapters", book.chapters);
        chap.setLong(book, 12);
        out.format(fmt, "after", "chapters", chap.getLong(book));

        Field chars = c.getDeclaredField("characters");
        out.format(fmt, "before", "characters",
               Arrays.asList(book.characters));
        String[] newChars = { "Queen", "King" };
        chars.set(book, newChars);
        out.format(fmt, "after", "characters",
               Arrays.asList(book.characters));

        Field t = c.getDeclaredField("twin");
        out.format(fmt, "before", "twin", book.twin);
        t.set(book, Tweedle.DUM);
        out.format(fmt, "after", "twin", t.get(book));

    } catch (NoSuchFieldException x) {
        x.printStackTrace();
    } catch (IllegalAccessException x) {
        x.printStackTrace();
    }
    }
}
```

通过反射进行赋值需要一定的性能消耗，用于前置操作，例如验证访问权限。从运行期的角度来看，结果是相同的，并且操作和在代码中修改的原子性相同。但是使用反射会失去虚拟机对代码的优化，下面就是个例子，虚拟机会优化，反射却不会。

```java
int x = 1;
x = 2;
x = 3;
```

### 需要注意的Exception

#### 不可转型异常 `IllegalArgumentException`

```java
import java.lang.reflect.Field;

public class FieldTrouble {
    public Integer val;

    public static void main(String... args) {
    FieldTrouble ft = new FieldTrouble();
    try {
        Class<?> c = ft.getClass();
        Field f = c.getDeclaredField("val");
        f.setInt(ft, 42); //IllegalArgumentException
    } catch (NoSuchFieldException x) {
        x.printStackTrace();
    } catch (IllegalAccessException x) {
        x.printStackTrace();
    }
    }
}

/**
 * RESULT:
 * $ java FieldTrouble
 * Exception in thread "main" java.lang.IllegalArgumentException: Can not set
 *  java.lang.Object field FieldTrouble.val to (long)42
 *        at sun.reflect.UnsafeFieldAccessorImpl.throwSetIllegalArgumentException
 *          (UnsafeFieldAccessorImpl.java:146)
 *        at sun.reflect.UnsafeFieldAccessorImpl.throwSetIllegalArgumentException
 *          (UnsafeFieldAccessorImpl.java:174)
 *        at sun.reflect.UnsafeObjectFieldAccessorImpl.setLong
 *          (UnsafeObjectFieldAccessorImpl.java:102)
 *        at java.lang.reflect.Field.setLong(Field.java:831)
 *        at FieldTrouble.main(FieldTrouble.java:11)
 */
```

* 对于非反射的表达式`Integer val = 42;`编译器会将值类型42装箱成包装类`new Integer(42)`。
* 当使用反射时，编译器不会进行装箱。

要解决上面的问题可以将`f.setInt(ft, 42);`改为调用`Field.set(Object obj, Object value)`方法。

#### 无权限修改异常

常见于修改private或final属性 `IllegalAccessException`


```java
import java.lang.reflect.Field;

public class FieldTroubleToo {
    public final boolean b = true;

    public static void main(String... args) {
    FieldTroubleToo ft = new FieldTroubleToo();
    try {
        Class<?> c = ft.getClass();
        Field f = c.getDeclaredField("b");
//      f.setAccessible(true);  // solution
        f.setBoolean(ft, Boolean.FALSE);   // IllegalAccessException
    } catch (NoSuchFieldException x) {
        x.printStackTrace();
    } catch (IllegalArgumentException x) {
        x.printStackTrace();
    } catch (IllegalAccessException x) {
        x.printStackTrace();
    }
    }
}
/**
 * $ java FieldTroubleToo
 * java.lang.IllegalAccessException: Can not set final boolean field
 *  FieldTroubleToo.b to (boolean)false
 *         at sun.reflect.UnsafeFieldAccessorImpl.
 *           throwFinalFieldIllegalAccessException(UnsafeFieldAccessorImpl.java:55)
 *         at sun.reflect.UnsafeFieldAccessorImpl.
 *           throwFinalFieldIllegalAccessException(UnsafeFieldAccessorImpl.java:63)
 *         at sun.reflect.UnsafeQualifiedBooleanFieldAccessorImpl.setBoolean
 *           (UnsafeQualifiedBooleanFieldAccessorImpl.java:78)
 *         at java.lang.reflect.Field.setBoolean(Field.java:686)
 *         at FieldTroubleToo.main(FieldTroubleToo.java:12)
 */
```
解决方案: 调用`f.setAccessible(true);`方法，提供访问权限.

## 反射中的类成员--Method

`java.lang.reflect.Method` 类提供了用于获取方法修饰符，返回类型，参数，注解和异常的API。同时，还可以通过API来调用方法。

### 获取方法声明信息

一个方法的声明包含了方法名，修饰符，参数，返回类型和可抛出的异常。下面是一个例子，展示了怎样列举出某个特定类的所有声明方法。

```java
import java.lang.reflect.Method;
import java.lang.reflect.Type;
import static java.lang.System.out;

public class MethodSpy {
    private static final String  fmt = "%24s: %s%n";

    // for the morbidly curious
    <E extends RuntimeException> void genericThrow() throws E {}

    public static void main(String... args) {
        try {
            Class<?> c = Class.forName(args[0]);
            Method[] allMethods = c.getDeclaredMethods();
            for (Method m : allMethods) {
            if (!m.getName().equals(args[1])) {
                continue;
            }
            out.format("%s%n", m.toGenericString());

            out.format(fmt, "ReturnType", m.getReturnType());
            out.format(fmt, "GenericReturnType", m.getGenericReturnType());

            Class<?>[] pType  = m.getParameterTypes();
            Type[] gpType = m.getGenericParameterTypes();
            for (int i = 0; i < pType.length; i++) {
                out.format(fmt,"ParameterType", pType[i]);
                out.format(fmt,"GenericParameterType", gpType[i]);
            }

            Class<?>[] xType  = m.getExceptionTypes();
            Type[] gxType = m.getGenericExceptionTypes();
            for (int i = 0; i < xType.length; i++) {
                out.format(fmt,"ExceptionType", xType[i]);
                out.format(fmt,"GenericExceptionType", gxType[i]);
            }
            }
        } catch (ClassNotFoundException x) {
            x.printStackTrace();
        }
    }
}
/**
 * $ java MethodSpy java.lang.Class getConstructor
 * public java.lang.reflect.Constructor<T> java.lang.Class.getConstructor
 *  (java.lang.Class<?>[]) throws java.lang.NoSuchMethodException,
 *  java.lang.SecurityException
 *              ReturnType: class java.lang.reflect.Constructor
 *       GenericReturnType: java.lang.reflect.Constructor<T>
 *           ParameterType: class [Ljava.lang.Class;
 *    GenericParameterType: java.lang.Class<?>[]
 *           ExceptionType: class java.lang.NoSuchMethodException
 *    GenericExceptionType: class java.lang.NoSuchMethodException
 *           ExceptionType: class java.lang.SecurityException
 *    GenericExceptionType: class java.lang.SecurityException
 */
```

> NOTE: `Method.getGenericExceptionTypes()`存在的原因是有可能使用泛型异常类型去声明一个方法，但是该方法很少被使用，因为捕获一个泛型异常是不可能的。
>>[官方说明](http://docs.oracle.com/javase/tutorial/java/generics/restrictions.html#cannotCatch)

### 获取方法参数

`java.lang.reflect.Executable.getParameters`方法可以获取方法或构造器的参数\(类`Method`和`Constructor`都继承了类`Executable`的`Executable.getParameters`方法\)。但是，class文件默认情况下不会存储形参的名称。

要将正式参数名称存储在class文件中，从而使Reflection API能够检索形参名称，请使用编译器选项 `-parameters`。

```java
/* 引用自 https://docs.oracle.com/javase/tutorial/reflect/member/example/MethodParameterSpy.java
*/
import java.lang.reflect.*;
import java.util.function.*;
import static java.lang.System.out;

public class MethodParameterSpy {

    private static final String  fmt = "%24s: %s%n";

    // for the morbidly curious
    <E extends RuntimeException> void genericThrow() throws E {}

    public static void printClassConstructors(Class c) {
        Constructor[] allConstructors = c.getConstructors();
        out.format(fmt, "Number of constructors", allConstructors.length);
        for (Constructor currentConstructor : allConstructors) {
            printConstructor(currentConstructor);
        }  
        Constructor[] allDeclConst = c.getDeclaredConstructors();
        out.format(fmt, "Number of declared constructors",
            allDeclConst.length);
        for (Constructor currentDeclConst : allDeclConst) {
            printConstructor(currentDeclConst);
        }
    }

    public static void printClassMethods(Class c) {
       Method[] allMethods = c.getDeclaredMethods();
        out.format(fmt, "Number of methods", allMethods.length);
        for (Method m : allMethods) {
            printMethod(m);
        }
    }

    public static void printConstructor(Constructor c) {
        out.format("%s%n", c.toGenericString());
        Parameter[] params = c.getParameters();
        out.format(fmt, "Number of parameters", params.length);
        for (int i = 0; i < params.length; i++) {
            printParameter(params[i]);
        }
    }

    public static void printMethod(Method m) {
        out.format("%s%n", m.toGenericString());
        out.format(fmt, "Return type", m.getReturnType());
        out.format(fmt, "Generic return type", m.getGenericReturnType());

        Parameter[] params = m.getParameters();
        for (int i = 0; i < params.length; i++) {
            printParameter(params[i]);
        }
    }

    public static void printParameter(Parameter p) {
        out.format(fmt, "Parameter class", p.getType());
        out.format(fmt, "Parameter name", p.getName());
        out.format(fmt, "Modifiers", p.getModifiers());
        out.format(fmt, "Is implicit?", p.isImplicit());
        out.format(fmt, "Is name present?", p.isNamePresent());
        out.format(fmt, "Is synthetic?", p.isSynthetic());
    }

    public static void main(String... args) {

        try {
            printClassConstructors(Class.forName(args[0]));
            printClassMethods(Class.forName(args[0]));
        } catch (ClassNotFoundException x) {
            x.printStackTrace();
        }
    }
}

import java.util.*; 
 
public class ExampleMethods<T> {
   
    public boolean simpleMethod(String stringParam, int intParam) {
        System.out.println("String: " + stringParam + ", integer: " + intParam); 
	    return true;
    }
    
    public int varArgsMethod(String... manyStrings) {
        return manyStrings.length;
    }
    
    public boolean methodWithList(List<String> listParam) {
        return listParam.isEmpty();
    }

    public <T> void genericMethod(T[] a, Collection<T> c) {
        System.out.println("Length of array: " + a.length);
        System.out.println("Size of collection: " + c.size()); 
    }   
}
```

执行 MethodParameterSpy 的输出:

```sh
$ java MethodParameterSpy ExampleMethods
Number of constructors: 1

Constructor #1
public ExampleMethods()

Number of declared constructors: 1

Declared constructor #1
public ExampleMethods()

Number of methods: 4

Method #1
public boolean ExampleMethods.simpleMethod(java.lang.String,int)
             Return type: boolean
     Generic return type: boolean
         Parameter class: class java.lang.String
          Parameter name: stringParam
               Modifiers: 0
            Is implicit?: false
        Is name present?: true
           Is synthetic?: false
         Parameter class: int
          Parameter name: intParam
               Modifiers: 0
            Is implicit?: false
        Is name present?: true
           Is synthetic?: false

Method #2
public int ExampleMethods.varArgsMethod(java.lang.String...)
             Return type: int
     Generic return type: int
         Parameter class: class [Ljava.lang.String;
          Parameter name: manyStrings
               Modifiers: 0
            Is implicit?: false
        Is name present?: true
           Is synthetic?: false

Method #3
public boolean ExampleMethods.methodWithList(java.util.List<java.lang.String>)
             Return type: boolean
     Generic return type: boolean
         Parameter class: interface java.util.List
          Parameter name: listParam
               Modifiers: 0
            Is implicit?: false
        Is name present?: true
           Is synthetic?: false

Method #4
public <T> void ExampleMethods.genericMethod(T[],java.util.Collection<T>)
             Return type: void
     Generic return type: void
         Parameter class: class [Ljava.lang.Object;
          Parameter name: a
               Modifiers: 0
            Is implicit?: false
        Is name present?: true
           Is synthetic?: false
         Parameter class: interface java.util.Collection
          Parameter name: c
               Modifiers: 0
            Is implicit?: false
        Is name present?: true
           Is synthetic?: false
```

### Parameter类中的方法

* getType：返回参数的声明类型的Class对象。
* getName：返回参数的名称。如果参数的名称存在，则此方法返回.class文件提供的名称。否则，以argN的形式返回，其中N是声明参数的方法中的参数索引。例如，假设您编译了ExampleMethods而未指定-parameters编译器选项。MethodParameterSpy将为方法ExampleMethods.simpleMethod打印以下内容：

```
public boolean ExampleMethods.simpleMethod(java.lang.String,int)
             Return type: boolean
     Generic return type: boolean
         Parameter class: class java.lang.String
          Parameter name: arg0
               Modifiers: 0
            Is implicit?: false
        Is name present?: false
           Is synthetic?: false
         Parameter class: int
          Parameter name: arg1
               Modifiers: 0
            Is implicit?: false
        Is name present?: false
           Is synthetic?: false
```
* getModifiers ：返回一个整数，表示形式参数拥有的各种特征。该值是以下值的总和：

|值（十进制）|值（十六进制）|描述|
|:---|:---|:---|
|16|0×0010|声明形式参数 final|
|4096|为0x1000|形式参数是合成的。或者，可以调用该方法isSynthetic。|
|32768|为0x8000|该参数在源代码中隐式声明。或者，您可以调用该方法isImplicit|

* isImplicit：如果在源代码中隐式声明此参数，则返回true。
* isNamePresent: 如果参数来自.class文件，则返回true。
* isSynthetic：如果在源代码中既未隐式声明也未显式声明此参数，则返回true。

### 隐式和合成参数

我们知道，在类未定义构造器时，JVM会自动生成默认构造器。在上面的输出中，可以看到:

```
Number of declared constructors: 1
public ExampleMethods()
```

考虑下面这种情况:

```java
public class MethodParameterExamples {
    public class InnerClass { }
}
```

InnerClass是非静态内部类。它的构造函数也是隐式声明的。但是，这个构造函数将包含一个参数。当Java编译器编译时InnerClass，它会创建一个类似于以下代码的.class文件：

```
public class MethodParameterExamples { 
    public class InnerClass { 
        final MethodParameterExamples parent; 
        InnerClass（final MethodParameterExamples this$0）{ 
            parent = this$0; 
        } 
    } 
}
```

这个内部类的构造器包含一个隐式参数，类型是封装类的类型，MethodParameterSpy将为方法打印:

```
public MethodParameterExamples$InnerClass(MethodParameterExamples)
         Parameter class: class MethodParameterExamples
          Parameter name: this$0
               Modifiers: 32784
            Is implicit?: true
        Is name present?: true
           Is synthetic?: false
```

java 编译器实现的构造器，如果在源代码中没有对应的声明方法(无论是显示声明还是隐式声明)，类初始化方法`<clinit>`除外。合成构造器是由java 编译器实现的，并且因为不同的java编译器实现，合成构造器也互相不同。以下面的类举例:

```java
public class MethodParameterExamples {
    enum Colors {
        RED, WHITE;
    }
}
```

当Java编译器遇到enum 的构造器时，它会生成一个`.class`文件和几个方法,以实现enum的功能。

```java
final class Colors extends java.lang.Enum<Colors> {
    public final static Colors RED = new Colors("RED", 0);
    public final static Colors BLUE = new Colors("WHITE", 1);
 
    private final static values = new Colors[]{ RED, BLUE };
 
    private Colors(String name, int ordinal) {
        super(name, ordinal);
    }
 
    public static Colors[] values(){
        return values;
    }
 
    public static Colors valueOf(String name){
        return (Colors)java.lang.Enum.valueOf(Colors.class, name);
    }
}
```

其中 values 和valueOf 是隐式方法,因此，它们的参数名称也是隐式声明的，` Colors(String name, int ordinal)`是默认构造函数，它是隐式声明的。但是其参数`(name, ordinal)`并不是隐式声明(默认构造器是无参的)，所以它们是合成参数。

> enum 类 默认构造函数的形式参数不是隐式声明的，因为不同的编译器不需要就此构造函数的形式达成一致;另一个 Java 编译器可能为其指定不同的形式参数。

MethodParameterSpy将为方法打印:

```
enum Colors:

Number of constructors: 0

Number of declared constructors: 1

Declared constructor #1
private MethodParameterExamples$Colors()
         Parameter class: class java.lang.String
          Parameter name: $enum$name
               Modifiers: 4096
            Is implicit?: false
        Is name present?: true
           Is synthetic?: true
         Parameter class: int
          Parameter name: $enum$ordinal
               Modifiers: 4096
            Is implicit?: false
        Is name present?: true
           Is synthetic?: true

Number of methods: 2

Method #1
public static MethodParameterExamples$Colors[]
    MethodParameterExamples$Colors.values()
             Return type: class [LMethodParameterExamples$Colors;
     Generic return type: class [LMethodParameterExamples$Colors;

Method #2
public static MethodParameterExamples$Colors
    MethodParameterExamples$Colors.valueOf(java.lang.String)
             Return type: class MethodParameterExamples$Colors
     Generic return type: class MethodParameterExamples$Colors
         Parameter class: class java.lang.String
          Parameter name: name
               Modifiers: 32768
            Is implicit?: true
        Is name present?: true
           Is synthetic?: false
```

### 分析方法修饰符


方法声明的几个修饰符：

* 访问修饰符：public, protected,private
* 静态修饰符：static
* 禁止修改值的修饰符：final
* 需要覆盖的修饰符：abstract
* 防止重入的修饰符：synchronized
* 指示在另一种编程语言中实现的修饰符：native
* 修改器强制严格的浮点行为：strictfp
* 注解：Annotations

下面例子列出了具有给定名称的方法的修饰符。它还显示该方法是合成的（编译器生成的）、可变参数 还是桥接方法（编译器生成的以支持泛型接口）。



```java
import java.lang.reflect.Method;
import java.lang.reflect.Modifier;
import static java.lang.System.out;

public class MethodModifierSpy {

    private static int count;
    private static synchronized void inc() { count++; }
    private static synchronized int cnt() { return count; }

    public static void main(String... args) {
	try {
	    Class<?> c = Class.forName(args[0]);
	    Method[] allMethods = c.getDeclaredMethods();
	    for (Method m : allMethods) {
		if (!m.getName().equals(args[1])) {
		    continue;
		}
		out.format("%s%n", m.toGenericString());
		out.format("  Modifiers:  %s%n",
			   Modifier.toString(m.getModifiers()));
		out.format("  [ synthetic=%-5b var_args=%-5b bridge=%-5b ]%n",
			   m.isSynthetic(), m.isVarArgs(), m.isBridge());
		inc();
	    }
	    out.format("%d matching overload%s found%n", cnt(),
		       (cnt() == 1 ? "" : "s"));

        // production code should handle this exception more gracefully
	} catch (ClassNotFoundException x) {
	    x.printStackTrace();
	}
    }
}
```

下面的方法为例:

```
$ java MethodModifierSpy java.lang.Object wait
public final void java.lang.Object.wait() throws java.lang.InterruptedException
  Modifiers:  public final
  [ synthetic=false var_args=false bridge=false ]
public final void java.lang.Object.wait(long,int)
  throws java.lang.InterruptedException
  Modifiers:  public final
  [ synthetic=false var_args=false bridge=false ]
public final native void java.lang.Object.wait(long)
  throws java.lang.InterruptedException
  Modifiers:  public final native
  [ synthetic=false var_args=false bridge=false ]
3 matching overloads found

$ java MethodModifierSpy java.lang.StrictMath toRadians
public static double java.lang.StrictMath.toRadians(double)
  Modifiers:  public static strictfp
  [ synthetic=false var_args=false bridge=false ]
1 matching overload found

$ java MethodModifierSpy MethodModifierSpy inc
private synchronized void MethodModifierSpy.inc()
  Modifiers: private synchronized
  [ synthetic=false var_args=false bridge=false ]
1 matching overload found

$ java MethodModifierSpy java.lang.Class getConstructor
public java.lang.reflect.Constructor<T> java.lang.Class.getConstructor
  (java.lang.Class<T>[]) throws java.lang.NoSuchMethodException,
  java.lang.SecurityException
  Modifiers: public transient
  [ synthetic=false var_args=true bridge=false ]
1 matching overload found

$ java MethodModifierSpy java.lang.String compareTo
public int java.lang.String.compareTo(java.lang.String)
  Modifiers: public
  [ synthetic=false var_args=false bridge=false ]
public int java.lang.String.compareTo(java.lang.Object)
  Modifiers: public volatile
  [ synthetic=true  var_args=false bridge=true  ]
2 matching overloads found
```

注意，Method.isVarArgs() 返回true是指如下结构:

```
public Constructor<T> getConstructor(Class<?>... parameterTypes)
```

而不是

```
public Constructor<T> getConstructor(Class<?> [] parameterTypes)
```

可以看到 String.compareTo() 包含了两个方法，在 String.java中声明的方法是:

```java
public int compareTo(String anotherString);
```
第二个合成方法是用于桥接的，发生这种情况是因为 String 实现了参数化接口 Comparable. 在类型擦除期间，继承的方法 Comparable.compareTo()的参数类型从 java.lang.Object更改为 java.lang.String 。由于在擦除后方法的参数类型不再匹配，因此无法进行重写。这将产生编译时错误，添加桥接方法可避免此问题.

Method实现了java.lang.reflect.AnnotatedElement。因此，任何带有java.lang.annotation.RetentionPolicy.RUNTIME的运行时注释都可以被检索。

### 调用方法

反射提供了一种在类上调用方法的方法。通常，仅当无法在非反射代码中将类的实例强制转换为所需类型时，才需要执行此操作。使用 java.lang.reflect.Method.invoke() 调用,第一个参数是要在其上调用此特定方法的对象实例(如果方法是 static，则第一个参数应为null).后续参数是方法的参数。如果方法调用引发异常，它将抛出 java.lang.reflect.InvocationTargetException 。可以使用 InvocationTargetException.getCause（） 方法检索该方法的原始异常。

```java
// 搜素所有以test开头的方法，且返回boolean值。参数类型为Locale
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.lang.reflect.Type;
import java.util.Locale;
import static java.lang.System.out;
import static java.lang.System.err;

public class Deet<T> {
    private boolean testDeet(Locale l) {
	// getISO3Language() may throw a MissingResourceException
	out.format("Locale = %s, ISO Language Code = %s%n", l.getDisplayName(), l.getISO3Language());
	return true;
    }

    private int testFoo(Locale l) { return 0; }
    private boolean testBar() { return true; }

    public static void main(String... args) {
	if (args.length != 4) {
	    err.format("Usage: java Deet <classname> <langauge> <country> <variant>%n");
	    return;
	}

	try {
	    Class<?> c = Class.forName(args[0]);
	    Object t = c.newInstance();

        //返回类中显式声明的所有方法
	    Method[] allMethods = c.getDeclaredMethods();
	    for (Method m : allMethods) {
		String mname = m.getName();
		if (!mname.startsWith("test")
		    || (m.getGenericReturnType() != boolean.class)) {
		    continue;
		}
 		Type[] pType = m.getGenericParameterTypes();
         // 确定定位方法的参数是否与所需的调用兼容
 		if ((pType.length != 1)
		    || Locale.class.isAssignableFrom(pType[0].getClass())) {
 		    continue;
 		}

		out.format("invoking %s()%n", mname);
		try {
		    m.setAccessible(true);
		    Object o = m.invoke(t, new Locale(args[1], args[2], args[3]));
		    out.format("%s() returned %b%n", mname, (Boolean) o);

		// Handle any exceptions thrown by method to be invoked.
		} catch (InvocationTargetException x) {
		    Throwable cause = x.getCause();
		    err.format("invocation of %s failed: %s%n",
			       mname, cause.getMessage());
		}
	    }

        // production code should handle these exceptions more gracefully
	} catch (ClassNotFoundException x) {
	    x.printStackTrace();
	} catch (InstantiationException x) {
	    x.printStackTrace();
	} catch (IllegalAccessException x) {
	    x.printStackTrace();
	}
    }
}
```

使用例子

```sh
$ java Deet Deet ja JP JP
invoking testDeet()
Locale = Japanese (Japan,JP), 
ISO Language Code = jpn
testDeet() returned true

$ java Deet Deet xx XX XX
invoking testDeet()
invocation of testDeet failed: 
Couldn't find 3-letter language code for xx
```

Method.invoke()可用于将可变数量的参数传递给方法。可变变量的方法的实现就像变量参数打包在数组中一样。

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.Arrays;

public class InvokeMain {
    public static void main(String... args) {
	try {
	    Class<?> c = Class.forName(args[0]);
	    Class[] argTypes = new Class[] { String[].class };
	    Method main = c.getDeclaredMethod("main", argTypes);
  	    String[] mainArgs = Arrays.copyOfRange(args, 1, args.length);
	    System.out.format("invoking %s.main()%n", c.getName());
	    main.invoke(null, (Object)mainArgs);

        // production code should handle these exceptions more gracefully
	} catch (ClassNotFoundException x) {
	    x.printStackTrace();
	} catch (NoSuchMethodException x) {
	    x.printStackTrace();
	} catch (IllegalAccessException x) {
	    x.printStackTrace();
	} catch (InvocationTargetException x) {
	    x.printStackTrace();
	}
    }
}
```

### Exception

#### 由于类型擦除引起的异常,NoSuchMethodException。

```java
import java.lang.reflect.Method;

public class MethodTrouble<T>  {
    public void lookup(T t) {}
    public void find(Integer i) {}

    public static void main(String... args) {
	try {
	    String mName = args[0];
	    Class cArg = Class.forName(args[1]);
	    Class<?> c = (new MethodTrouble<Integer>()).getClass();
	    Method m = c.getMethod(mName, cArg);
	    System.out.format("Found:%n  %s%n", m.toGenericString());

        // production code should handle these exceptions more gracefully
	} catch (NoSuchMethodException x) {
	    x.printStackTrace();
	} catch (ClassNotFoundException x) {
	    x.printStackTrace();
	}
    }
}
//-----------------------------------
$ java MethodTrouble lookup java.lang.Integer
java.lang.NoSuchMethodException: MethodTrouble.lookup(java.lang.Integer)
        at java.lang.Class.getMethod(Class.java:1605)
        at MethodTrouble.main(MethodTrouble.java:12)
$ java MethodTrouble lookup java.lang.Object
Found:
  public void MethodTrouble.lookup(T)
```

当使用泛型参数类型声明方法时，编译器会将泛型类型替换为其上限，在本例中，上限为 Object。因此，当代码搜索 时，找不到任何lookup方法.

#### 非法访问产生的IllegalAccessException

如果尝试调用私有或以其他方式无法访问的方法，会产生异常IllegalAccessException

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

class AnotherClass {
    private void m() {}
}

public class MethodTroubleAgain {
    public static void main(String... args) {
	AnotherClass ac = new AnotherClass();
	try {
	    Class<?> c = ac.getClass();
 	    Method m = c.getDeclaredMethod("m");
         // 通过 AccessibleObject.setAccessible()禁止此检查的功能。
//  	    m.setAccessible(true);      // solution
 	    Object o = m.invoke(ac);    // IllegalAccessException

        // production code should handle these exceptions more gracefully
	} catch (NoSuchMethodException x) {
	    x.printStackTrace();
	} catch (InvocationTargetException x) {
	    x.printStackTrace();
	} catch (IllegalAccessException x) {
	    x.printStackTrace();
	}
    }
}
//-----------------------------------
$ java MethodTroubleAgain
java.lang.IllegalAccessException: Class MethodTroubleAgain can not access a
  member of class AnotherClass with modifiers "private"
        at sun.reflect.Reflection.ensureMemberAccess(Reflection.java:65)
        at java.lang.reflect.Method.invoke(Method.java:588)
        at MethodTroubleAgain.main(MethodTroubleAgain.java:15)
```

#### IllegalArgumentException

```java
import java.lang.reflect.Method;

public class MethodTroubleToo {
    public void ping() { System.out.format("PONG!%n"); }

    public static void main(String... args) {
	try {
	    MethodTroubleToo mtt = new MethodTroubleToo();
	    Method m = MethodTroubleToo.class.getMethod("ping");

 	    switch(Integer.parseInt(args[0])) {
	    case 0:
  		m.invoke(mtt);                 // works
		break;
	    case 1:
 		m.invoke(mtt, null);           // works (expect compiler warning)
		break;
	    case 2:
		Object arg2 = null;
		m.invoke(mtt, arg2);           // IllegalArgumentException
		break;
	    case 3:
		m.invoke(mtt, new Object[0]);  // works
		break;
	    case 4:
		Object arg4 = new Object[0];
		m.invoke(mtt, arg4);           // IllegalArgumentException
		break;
	    default:
		System.out.format("Test not found%n");
	    }

        // production code should handle these exceptions more gracefully
	} catch (Exception x) {
	    x.printStackTrace();
	}
    }
}
$ java MethodTroubleToo 0
PONG!
```

由于 Method.invoke() 的所有参数（第一个参数除外）都是可选的，因此当要调用的方法没有参数时，可以省略它们。

```java
$ java MethodTroubleToo 1
PONG!
```

在这种情况下，代码会生成此编译器警告，因为null是不明确的。

```
$ javac MethodTroubleToo.java
MethodTroubleToo.java:16: warning: non-varargs call of varargs method with
  inexact argument type for last parameter;
 		m.invoke(mtt, null);           // works (expect compiler warning)
 		              ^
  cast to Object for a varargs call
  cast to Object[] for a non-varargs call and to suppress this warning
1 warning
```

无法确定null是表示空参数数组还是 第一个参数.

```java
$ java MethodTroubleToo 2
java.lang.IllegalArgumentException: wrong number of arguments
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke
          (NativeMethodAccessorImpl.java:39)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke
          (DelegatingMethodAccessorImpl.java:25)
        at java.lang.reflect.Method.invoke(Method.java:597)
        at MethodTroubleToo.main(MethodTroubleToo.java:21)
```

尽管参数是null ，但这个还是失败，是因为该类型是一个 Object，并且ping()根本不需要任何参数。

```java
$ java MethodTroubleToo 3
PONG!
```

这之所以有效，是因为创建一个空数组 `new Object[0]`，这等效于不传递任何可选参数。

```java
$ java MethodTroubleToo 4
java.lang.IllegalArgumentException: wrong number of arguments
        at sun.reflect.NativeMethodAccessorImpl.invoke0
          (Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke
          (NativeMethodAccessorImpl.java:39)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke
          (DelegatingMethodAccessorImpl.java:25)
        at java.lang.reflect.Method.invoke(Method.java:597)
        at MethodTroubleToo.main(MethodTroubleToo.java:28)
```

与前面的示例不同，如果空数组存储在 Object 中，则将其视为 Object。这与案例 2 失败的原因相同，ping()不需要参数。

#### InvocationTargetException 

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class MethodTroubleReturns {
    private void drinkMe(int liters) {
	if (liters < 0)
	    throw new IllegalArgumentException("I can't drink a negative amount of liquid");
    }

    public static void main(String... args) {
	try {
	    MethodTroubleReturns mtr  = new MethodTroubleReturns();
 	    Class<?> c = mtr.getClass();
   	    Method m = c.getDeclaredMethod("drinkMe", int.class);
	    m.invoke(mtr, -1);

        // production code should handle these exceptions more gracefully
	} catch (InvocationTargetException x) {
	    Throwable cause = x.getCause();
	    System.err.format("drinkMe() failed: %s%n", cause.getMessage());
	} catch (Exception x) {
	    x.printStackTrace();
	}
    }
}
$ java MethodTroubleReturns
drinkMe() failed: I can't drink a negative amount of liquid
```

## 反射中的类成员--Constructor

`java.lang.reflect.Constructor` 表示类的构造器。

### 查找构造函数

示例说明了如何在类的声明构造函数中搜索具有给定类型参数的构造函数

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Type;
import static java.lang.System.out;

public class ConstructorSift {
    public static void main(String... args) {
	try {
	    Class<?> cArg = Class.forName(args[1]);

	    Class<?> c = Class.forName(args[0]);
	    Constructor[] allConstructors = c.getDeclaredConstructors();
	    for (Constructor ctor : allConstructors) {
		Class<?>[] pType  = ctor.getParameterTypes();
		for (int i = 0; i < pType.length; i++) {
		    if (pType[i].equals(cArg)) {
			out.format("%s%n", ctor.toGenericString());

			Type[] gpType = ctor.getGenericParameterTypes();
			for (int j = 0; j < gpType.length; j++) {
			    char ch = (pType[j].equals(cArg) ? '*' : ' ');
			    out.format("%7c%s[%d]: %s%n", ch,
				       "GenericParameterType", j, gpType[j]);
			}
			break;
		    }
		}
	    }

        // production code should handle this exception more gracefully
	} catch (ClassNotFoundException x) {
	    x.printStackTrace();
	}
    }
}
```

Method.getGenericParameterTypes()将查阅类文件中的签名属性（如果存在）。如果该属性不可用，它将回退到 Method.getParameterType(). 该类型未因泛型的引入而更改.

### 分析构造函数修饰符

构造函数的修饰符较少:

* 访问修饰符：public,protected,private
* 注解：Annotations

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Modifier;
import java.lang.reflect.Type;
import static java.lang.System.out;

public class ConstructorAccess {
    public static void main(String... args) {
	try {
	    Class<?> c = Class.forName(args[0]);
	    Constructor[] allConstructors = c.getDeclaredConstructors();
	    for (Constructor ctor : allConstructors) {
		int searchMod = modifierFromString(args[1]);
		int mods = accessModifiers(ctor.getModifiers());
		if (searchMod == mods) {
		    out.format("%s%n", ctor.toGenericString());
		    out.format("  [ synthetic=%-5b var_args=%-5b ]%n",
			       ctor.isSynthetic(), ctor.isVarArgs());
		}
	    }

        // production code should handle this exception more gracefully
	} catch (ClassNotFoundException x) {
	    x.printStackTrace();
	}
    }

    private static int accessModifiers(int m) {
	return m & (Modifier.PUBLIC | Modifier.PRIVATE | Modifier.PROTECTED);
    }

    private static int modifierFromString(String s) {
	if ("public".equals(s))               return Modifier.PUBLIC;
	else if ("protected".equals(s))       return Modifier.PROTECTED;
	else if ("private".equals(s))         return Modifier.PRIVATE;
	else if ("package-private".equals(s)) return 0;
	else return -1;
    }
}
// -----------------
$ java ConstructorAccess java.io.File private
private java.io.File(java.lang.String,int)
  [ synthetic=false var_args=false ]
private java.io.File(java.lang.String,java.io.File)
  [ synthetic=false var_args=false ]

```

### 创建新的类实例

有两种反射方法可用于创建类的实例：java.lang.reflect.Constructor.newInstance() 和 Class.newInstance() 。前者是优选的，因为：

* Class.newInstance() 只能调用零参数构造函数，而 Constructor.newInstance()可以调用任何构造函数，而不管参数的数量如何。
* Class.newInstance()  会引发构造函数引发的任何异常，无论它是检查还是未检查。Constructor.newInstance()总是用 InvocationTargetException 包装抛出的异常。
* Class.newInstance()  要求构造函数是可见的;Constructor.newInstance()在某些情况下可以调用private构造函数。

```java
import java.io.Console;
import java.nio.charset.Charset;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import static java.lang.System.out;

public class ConsoleCharset {
    public static void main(String... args) {
	Constructor[] ctors = Console.class.getDeclaredConstructors();
	Constructor ctor = null;
	for (int i = 0; i < ctors.length; i++) {
	    ctor = ctors[i];
	    if (ctor.getGenericParameterTypes().length == 0)
		break;
	}

	try {
	    ctor.setAccessible(true);
 	    Console c = (Console)ctor.newInstance();
	    Field f = c.getClass().getDeclaredField("cs");
	    f.setAccessible(true);
	    out.format("Console charset         :  %s%n", f.get(c));
	    out.format("Charset.defaultCharset():  %s%n",
		       Charset.defaultCharset());

        // production code should handle these exceptions more gracefully
	} catch (InstantiationException x) {
	    x.printStackTrace();
 	} catch (InvocationTargetException x) {
 	    x.printStackTrace();
	} catch (IllegalAccessException x) {
	    x.printStackTrace();
	} catch (NoSuchFieldException x) {
	    x.printStackTrace();
	}
    }
}
```

UNIX 系统的示例输出：

```
$ java ConsoleCharset
Console charset          :  ISO-8859-1
Charset.defaultCharset() :  ISO-8859-1
```

Constructor.newInstance() 的另一个常见应用是调用采用参数的构造函数.

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.util.HashMap;
import java.util.Map;
import java.util.Set;
import static java.lang.System.out;

class EmailAliases {
    private Set<String> aliases;
    private EmailAliases(HashMap<String, String> h) {
	aliases = h.keySet();
    }

    public void printKeys() {
	out.format("Mail keys:%n");
	for (String k : aliases)
	    out.format("  %s%n", k);
    }
}

public class RestoreAliases {

    private static Map<String, String> defaultAliases = new HashMap<String, String>();
    static {
	defaultAliases.put("Duke", "duke@i-love-java");
	defaultAliases.put("Fang", "fang@evil-jealous-twin");
    }

    public static void main(String... args) {
	try {
	    Constructor ctor = EmailAliases.class.getDeclaredConstructor(HashMap.class);
	    ctor.setAccessible(true);
	    EmailAliases email = (EmailAliases)ctor.newInstance(defaultAliases);
	    email.printKeys();

        // production code should handle these exceptions more gracefully
	} catch (InstantiationException x) {
	    x.printStackTrace();
	} catch (IllegalAccessException x) {
	    x.printStackTrace();
	} catch (InvocationTargetException x) {
	    x.printStackTrace();
	} catch (NoSuchMethodException x) {
	    x.printStackTrace();
	}
    }
}
// -----------------------------------------
$ java RestoreAliases
Mail keys:
  Duke
  Fang
```


## Array

数组类型可以通过调用 Class.isArray()来标识。反射支持通过java.lang.reflect.Array.newInstance()动态创建任意类型和维度的数组的能力.

```java
// 一个能够动态创建数组的基本解释器。将解析的语法如下：

//fully_qualified_class_name variable_name[] = 
//     { val1, val2, val3, ... }
import java.lang.reflect.Array;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.util.regex.Pattern;
import java.util.regex.Matcher;
import java.util.Arrays;
import static java.lang.System.out;

public class ArrayCreator {
    private static String s = "java.math.BigInteger bi[] = { 123, 234, 345 }";
    private static Pattern p = Pattern.compile("^\\s*(\\S+)\\s*\\w+\\[\\].*\\{\\s*([^}]+)\\s*\\}");

    public static void main(String... args) {
        Matcher m = p.matcher(s);

        if (m.find()) {
            String cName = m.group(1);
            String[] cVals = m.group(2).split("[\\s,]+");
            int n = cVals.length;

            try {
                Class<?> c = Class.forName(cName);
                Object o = Array.newInstance(c, n);
                for (int i = 0; i < n; i++) {
                    String v = cVals[i];
                    Constructor ctor = c.getConstructor(String.class);
                    Object val = ctor.newInstance(v);
                    Array.set(o, i, val);
                }

                Object[] oo = (Object[])o;
                out.format("%s[] = %s%n", cName, Arrays.toString(oo));

            // production code should handle these exceptions more gracefully
            } catch (ClassNotFoundException x) {
                x.printStackTrace();
            } catch (NoSuchMethodException x) {
                x.printStackTrace();
            } catch (IllegalAccessException x) {
                x.printStackTrace();
            } catch (InstantiationException x) {
                x.printStackTrace();
            } catch (InvocationTargetException x) {
                x.printStackTrace();
            }
        }
    }
}
$ java ArrayCreator
java.math.BigInteger [] = [123, 234, 345]
```

### 数组的操作

* java.lang.reflect.Field.set(Object obj，Object value) : 设置数组的值
* java.lang.reflect.Field.get(Object): 获取数组的值
* java.lang.reflect.Array 提供了对数组内部操作的方法

```java
import java.lang.reflect.Array;
import static java.lang.System.out;

public class CreateMatrix {
    public static void main(String... args) {
        Object matrix = Array.newInstance(int.class, 2, 2);
        Object row0 = Array.get(matrix, 0);
        Object row1 = Array.get(matrix, 1);

        Array.setInt(row0, 0, 1);
        Array.setInt(row0, 1, 2);
        Array.setInt(row1, 0, 3);
        Array.setInt(row1, 1, 4);

        for (int i = 0; i < 2; i++)
            for (int j = 0; j < 2; j++)
                out.format("matrix[%d][%d] = %d%n", i, j, ((int[][])matrix)[i][j]);
    }
}
$ java CreateMatrix
matrix[0][0] = 1
matrix[0][1] = 2
matrix[1][0] = 3
matrix[1][1] = 4
```

>   Array.newInstance(Class<？> componentType， int... dimensions) 提供了一种创建多维数组的便捷方法

## enum

- Class.isEnum() 指示此类是否表示枚举类型
- Class.getEnumConstants() 检索枚举常量的列表，这些枚举常量按声明的顺序进行定义,返回的值与调用枚举类型values()返回的值相同。
- java.lang.reflect.Field.isEnumConstant() 指示此字段是否表示枚举类型的元素

```java
import java.util.Arrays;
import static java.lang.System.out;

enum Eon { HADEAN, ARCHAEAN, PROTEROZOIC, PHANEROZOIC }

public class EnumConstants {
    public static void main(String... args) {
	try {
	    Class<?> c = (args.length == 0 ? Eon.class : Class.forName(args[0]));
	    out.format("Enum name:  %s%nEnum constants:  %s%n",
		       c.getName(), Arrays.asList(c.getEnumConstants()));
	    if (c == Eon.class)
		out.format("  Eon.values():  %s%n",
			   Arrays.asList(Eon.values()));

        // production code should handle this exception more gracefully
	} catch (ClassNotFoundException x) {
	    x.printStackTrace();
	}
    }
}
// ---------------------
$ java EnumConstants java.lang.annotation.RetentionPolicy
Enum name:  java.lang.annotation.RetentionPolicy
Enum constants:  [SOURCE, CLASS, RUNTIME]
$ java EnumConstants java.util.concurrent.TimeUnit
Enum name:  java.util.concurrent.TimeUnit
Enum constants:  [NANOSECONDS, MICROSECONDS, 
                  MILLISECONDS, SECONDS, 
                  MINUTES, HOURS, DAYS]
```

由于enum实际上是类，可以用上面的反射方法获取 字段、方法和构造函数。

```java
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.lang.reflect.Member;
import java.util.List;
import java.util.ArrayList;
import static java.lang.System.out;

public class EnumSpy {
    private static final String fmt = "  %11s:  %s %s%n";

    public static void main(String... args) {
	try {
	    Class<?> c = Class.forName(args[0]);
	    if (!c.isEnum()) {
		out.format("%s is not an enum type%n", c);
		return;
	    }
	    out.format("Class:  %s%n", c);

	    Field[] flds = c.getDeclaredFields();
	    List<Field> cst = new ArrayList<Field>();  // enum constants
	    List<Field> mbr = new ArrayList<Field>();  // member fields
	    for (Field f : flds) {
		if (f.isEnumConstant())
		    cst.add(f);
		else
		    mbr.add(f);
	    }
	    if (!cst.isEmpty())
		print(cst, "Constant");
	    if (!mbr.isEmpty())
		print(mbr, "Field");

	    Constructor[] ctors = c.getDeclaredConstructors();
	    for (Constructor ctor : ctors) {
		out.format(fmt, "Constructor", ctor.toGenericString(),
			   synthetic(ctor));
	    }

	    Method[] mths = c.getDeclaredMethods();
	    for (Method m : mths) {
		out.format(fmt, "Method", m.toGenericString(),
			   synthetic(m));
	    }

        // production code should handle this exception more gracefully
	} catch (ClassNotFoundException x) {
	    x.printStackTrace();
	}
    }

    private static void print(List<Field> lst, String s) {
	for (Field f : lst) {
 	    out.format(fmt, s, f.toGenericString(), synthetic(f));
	}
    }

    private static String synthetic(Member m) {
	return (m.isSynthetic() ? "[ synthetic ]" : "");
    }
}
$ java EnumSpy java.lang.annotation.RetentionPolicy
Class:  class java.lang.annotation.RetentionPolicy
     Constant:  public static final java.lang.annotation.RetentionPolicy
                  java.lang.annotation.RetentionPolicy.SOURCE 
     Constant:  public static final java.lang.annotation.RetentionPolicy
                  java.lang.annotation.RetentionPolicy.CLASS 
     Constant:  public static final java.lang.annotation.RetentionPolicy 
                  java.lang.annotation.RetentionPolicy.RUNTIME 
        Field:  private static final java.lang.annotation.RetentionPolicy[] 
                  java.lang.annotation.RetentionPolicy. [ synthetic ]
  Constructor:  private java.lang.annotation.RetentionPolicy() 
       Method:  public static java.lang.annotation.RetentionPolicy[]
                  java.lang.annotation.RetentionPolicy.values() 
       Method:  public static java.lang.annotation.RetentionPolicy
                  java.lang.annotation.RetentionPolicy.valueOf(java.lang.String) 
```

> Class.getFields()和 Class.getDeclaredFields() 不保证返回值的顺序与声明源代码中的顺序匹配。如果应用程序需要排序，请使用 Class.getEnumConstants()

* java.lang.reflect.Field.set(Object obj，Object value) : 设置enum的值
* java.lang.reflect.Field.get(Object): 获取enum的值

```java
import java.lang.reflect.Field;
import static java.lang.System.out;

enum TraceLevel { OFF, LOW, MEDIUM, HIGH, DEBUG }

class MyServer {
    private TraceLevel level = TraceLevel.OFF;
}

public class SetTrace {
    public static void main(String... args) {
	TraceLevel newLevel = TraceLevel.valueOf(args[0]);

	try {
	    MyServer svr = new MyServer();
	    Class<?> c = svr.getClass();
	    Field f = c.getDeclaredField("level");
	    f.setAccessible(true);
	    TraceLevel oldLevel = (TraceLevel)f.get(svr);
	    out.format("Original trace level:  %s%n", oldLevel);

	    if (oldLevel != newLevel) {
 		f.set(svr, newLevel);
		out.format("    New  trace level:  %s%n", f.get(svr));
	    }

        // production code should handle these exceptions more gracefully
	} catch (IllegalArgumentException x) {
	    x.printStackTrace();
	} catch (IllegalAccessException x) {
	    x.printStackTrace();
	} catch (NoSuchFieldException x) {
	    x.printStackTrace();
	}
    }
}
```

