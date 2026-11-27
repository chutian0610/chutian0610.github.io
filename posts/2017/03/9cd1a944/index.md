# Java序列化

"持久化"意味着对象的“生存时间”并不取决于程序是否正在执行——它存活于程序的每一次调用之间。通过序列化一个对象，将其写入磁盘，以后在程序再次调用时,通过反序列化,重新恢复那个对象，就能圆满实现一种“持久”效果。什么是序列化和反序列化？(注意这里仅指 java 自身提供的序列化功能)

* 把对象转换为字节序列的过程称为对象的序列化。
* 把字节序列恢复为对象的过程称为对象的反序列化。

<!--more-->

## 通过对象流进行序列化

序列化中对象则必须实现`Serializable`接口或是`Externalizable`接口。

```java
public class User implements Serializable {
    private String userName;
    private String password;
    private Integer id;

    public void setUserName(String userName) {
        this.userName = userName;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    @Override
    public String toString() {
        return "User{" +
                "userName='" + userName + '\'' +
                ", password='" + password + '\'' +
                ", id=" + id +
                '}';
    }
}
```

Java IO 包中为我们提供了 ObjectInputStream 和 ObjectOutputStream 两个类。java.io.ObjectOutputStream 类实现object的序列化功能。java.io.ObjectInputStream 类实现了object的反序列化功能。

```java
// 这是对上面 User 序列化的例子
public static void main(String[] args) throws IOException, ClassNotFoundException {
        String path = System.getProperty("user.dir");
        File file = new File(path+File.separator+"user.data");
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream(file));
        User user1 = new User();
        user1.setUserName("XXX");
        user1.setPassword("###");
        user1.setId(1);
        oos.writeObject(user1); // 序列化到字节
        oos.flush();            // 强制刷新缓冲区

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream(file));
        User user2 = (User)ois.readObject(); // 从字节反序列化
        System.out.println(user2);
}
//out >> User{userName='XXX', password='###', id=1}
```

`ObjectOutputStream.writeObject(Object object)` 方法，提供了JAVA默认的序列化方案:序列化传入参数，并以递归的方式遍历对象依赖图中的其他对象。用于创建完整的序列化表示。`ObjectOutputStream.readObject()` 方法反序列化流中的对象，并以递归的方式遍历对其他对象的引用，用于创建完整的对象。

在上述的遍历图的过程中，如果出现一个对象没有实现序列化接口，将会触发NotSerializableException。

## 序列化和继承

若父类实现序列化，则子类自动实现序列化。可序列化类的所有子类型本身可序列化，序列化接口没有方法或字段并且仅用于标识可序列化的语义。为了允许序列化序列化类的非序列化父类型，父类型必须要有无参构造器，方便子类处理父类属性(使用父类属性的空值)。如果不是这样,会在运行时抛出这个错误。

```java
Exception in thread "main" java.io.InvalidClassException: XXXX; no valid constructor
	at java.io.ObjectStreamClass$ExceptionInfo.newInvalidClassException(ObjectStreamClass.java:169)
	at java.io.ObjectStreamClass.checkDeserialize(ObjectStreamClass.java:874)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2043)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1573)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:431)
```

由于多种原因，强烈建议不要对内部类（即非静态成员类的嵌套类）（包括本地类和匿名类）进行序列化。因为在非静态上下文中声明的内部类包含对封闭类实例的隐式非瞬态引用，所以序列化这样的内部类实例也将导致其关联的外部类实例的序列化。由javac（或其他Java 编译器）生成的用于实现内部类的合成字段是依赖于实现的，并且可能在编译器之间变化; 

这些字段的差异可能会破坏兼容性，并导致违约冲突serialVersionUID值。分配给本地和匿名内部类的名称也依赖于实现，并且编译器之间可能不同。由于内部类不能声明除编译时常量字段之外的静态成员，因此它们不能使用该 serialPersistentFields机制来指定可序列化字段。最后，因为与外部实例关联的内部类没有零参数构造函数（此类内部类的构造函数隐式接受封闭实例作为前置参数），所以它们无法实现 Externalizable。但是，上面列出的所有问题都不适用于静态成员类。

## 序列化版本

每个序列化类都有一个版本标识，这个标识用于判断流中的对象是否是当前类的序列化结果，以及是否可以反序列化。如果没有为类声明serialVersionUID，则该值默认为该类的哈希值，是类名，接口类名，方法和字段的64位散列。

```java
private static final long serialVersionUID = 3487495895819393L;
```

当我们新增了User类的属性 `private String addr;`,那么如果我们没有指定serialVersionUID，在反序列化前一个版本生成的序列化文件时，会使用默认生成的serialVersionUID，从而发生异常:

```java
Exception in thread "main" java.io.InvalidClassException: XXXX; local class incompatible: stream classdesc serialVersionUID = 3487495895819393, local class serialVersionUID = 2144810805526923675
	at java.io.ObjectStreamClass.initNonProxy(ObjectStreamClass.java:687)
	at java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:1876)
	at java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1745)
	at java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2033)
	at java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1567)
	at java.io.ObjectInputStream.readObject(ObjectInputStream.java:427)
```

在上面的异常中，明确指出异常在于对象流中的serialVersionUID是3487495895819393，而本地serialVersionUID是2144810805526923675，这会导致验证不过。当然，如果我们指定serialVersionUID是-1，那么就不会产生异常，但是这样做，会导致版本管理失去作用。例如，添加字段，和删除字段，会丢失属性和初始化null 属性。

强烈建议所有可序列化类显式声明 serialVersionUID值，因为默认 serialVersionUID计算对类详细信息高度敏感，这些详细信息可能因编译器实现而异，因此可能在serialVersionUID 反序列化期间导致意外冲突，从而导致反序列化失败。

## 序列化字段管理

如果考虑安全问题，我们不想把密码序列化进行保存，那么该怎么做呢？这里有两个解决方法。

第一种方法是使用transient 关键字,当一个属性被声明为transient后，默认序列化机制就会忽略该字段。

```java
public class User implements Serializable {

    private String userName;
    private transient String password;
    private Integer id;
}
// User{userName='XXX', password='null', id=1}
```

在将 password 字段设为 transient后，可以看到序列化的结果中是不含password值的。

第二种方法是为类设置静态字段serialPersistentFields,必须使用ObjectStreamField列出可序列化字段的名称和类型,并把它写入对象数组serialPersistentFields。该字段的修饰符必须是private，static和final。如果字段的值为null或者不是实例 ObjectStreamField[]，或者字段没有所需的修饰符，则等同于没有设置。

```java
public class User implements Serializable {
    private static final ObjectStreamField [] serialPersistentFields = {
            new ObjectStreamField("userName",String.class),
            new ObjectStreamField("id",Integer.class)
    };

    private String userName;
    private String password;
    private Integer id;
}
// User{userName='XXX', password='null', id=1}
```

假设同时使用了 transient 和serialPersistentFields,会发生什么?

**transient会被忽略,只使用serialPersistentFields的设置**。

```java
public class User implements Serializable {
    private static final ObjectStreamField [] serialPersistentFields = {
            new ObjectStreamField("userName",String.class),
            new ObjectStreamField("password",String.class),
            new ObjectStreamField("id",Integer.class)
    };

    private String userName;
    private transient String password;
    private Integer id;
}
// User{userName='XXX', password='###', id=1}
```

## 序列化的扩展方法

假设，序列化需要保存密码，同时需要加密，那么该怎么办？

### Serializable 对象

Java 提供了在序列化期间的自定义实现，对于Serializable对象，writeObject方法允许类控制其自己字段的序列化。readObject方法允许类控制其自己字段的反序列化。

```java
private void writeObject(ObjectOutputStream stream) throws IOException;
private void readObject(ObjectInputStream stream) throws IOException, ClassNotFoundException;
```

使用时有几点需要注意下：

1. writeObject，readObject 是Serializable子类自己实现的方法，如果没有实现这些方法，默认是使用ObjectOutputStream的defaultWriteObject和ObjectInputStream的defaultReadObject方法来提供默认的序列化和反序列化。以writeObject为例:

```java
ObjectOutputStream.writeObject()
->ObjectOutputStream.writeObject0
->ObjectOutputStream.writeOrdinaryObject()
->ObjectOutputStream.writeSerialData()
->ObjectStreamClass.invokeWriteObject()
```

2. 如果实现了这两个方法，那么类的状态将由该方法托管，需要注意的是方法仅负责编写类自己的字段，而不是其超类型或子类型的字段。下面是ArrayList的例子(注意write/readObject中的顺序)

```java
//ArrayList
 private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();
        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);
        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }
        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
    /**
     * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
     * deserialize it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;
        // Read in size, and any hidden stuff
        s.defaultReadObject();
        // Read in capacity
        s.readInt(); // ignored
        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);
            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }

    }
```

在使用writeObject和readObject的时候，常常还会使用到ObjectOutputStream.PutField和ObjectOutputStream.GetField,用于按照Key-value的方式写入字段，优势是可以忽略字段顺序。

```java
// StringBuffer.java
 private synchronized void writeObject(ObjectOutputStream out)
            throws IOException {
        ObjectOutputStream.PutField fields = out.putFields();
        fields.put("count", length());
        fields.put("shared", false);
        fields.put("value", getValue());
        out.writeFields();
    }
 
    private void readObject(ObjectInputStream in) throws IOException,
            ClassNotFoundException {
        ObjectInputStream.GetField fields = in.readFields();
        int count = fields.get("count", 0);
        char[] value = (char[]) fields.get("value", null);
        set(value, count);
}
```

除了自定义扩展方法外,对于Serializable对象还可以使用writeReplace和readResolve方法来实现拦截。

* 当对象声明并实现了writeReplace后，序列化的对象其实是writeReplace的返回值。
* 当对象声明并实现了readResolve后，反序列化的对象其实是readResolve的返回值。

### Externalizable 对象

Externalizable继承自Serializable。

```java
public interface Externalizable extends Serializable
{
    public void writeExternal(ObjectOutput out)
        throws IOException;

    public void readExternal(ObjectInput in)
        throws IOException, java.lang.ClassNotFoundException;
}
```

1. 使用Externalizable接口需要实现writeExternal以及readExternal方法。
2. Externalizable接口的实现方式一定要有默认的无参构造函数(见ObjectStreamClass的构造器)

```java
   if (externalizable) {
                        cons = getExternalizableConstructor(cl);
    } else {
                        cons = getSerializableConstructor(cl); 
    ...

    /**
     * Returns public no-arg constructor of given class, or null if none found.
     * Access checks are disabled on the returned constructor (if any), since
     * the defining class may still be non-public.
     */
    private static Constructor<?> getExternalizableConstructor(Class<?> cl) {
        try {
            Constructor<?> cons = cl.getDeclaredConstructor((Class<?>[]) null);
            cons.setAccessible(true);
            return ((cons.getModifiers() & Modifier.PUBLIC) != 0) ?
                cons : null;
        } catch (NoSuchMethodException ex) {
            return null;
        }
    }
```
3. 采用Externalizable无需产生序列化ID（serialVersionUID）

## 反序列化后的认证

JDK 提供了ObjectInputValidation接口，用于反序列化后检验对象的有效性。

```java
public interface ObjectInputValidation
{
    public void validateObject()
        throws InvalidObjectException;
}

public class ValidateMe implements Serializable, ObjectInputValidation {
    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.registerValidation(this, 0);
        in.defaultReadObject();
    }

    public void validateObject() throws InvalidObjectException { 
        // if (this object is not valid) 
        throw new InvalidObjectException("Object not valid!"); 
    }
} 
```

## 枚举类的序列化

枚举常量的序列化与普通的可序列化或可外部化的对象不同。枚举常量的序列化形式仅由其名称组成；常量的字段值不存在于表单中。要序列化枚举常量，请ObjectOutputStream 写入由枚举常量的name方法返回的值 。要反序列化枚举常量，请 ObjectInputStream从流中读取常量名称。然后通过调用该java.lang.Enum.valueOf方法，将常量的枚举类型与接收到的常量名称作为参数传递，来获得反序列化的常量 。

枚举常数被序列过程中不能被定制：任何由枚举类型定义的方法writeObject,readObject,readObjectNoData，writeReplace和readResolve被序列化和反序列化期间被忽略。同样，任何 serialPersistentFields或 serialVersionUID字段声明也将被忽略,所有枚举类型都有一个固定serialVersionUID,0L。不需要为枚举类型记录可序列化的字段和数据，因为发送的数据类型没有变化。

## 序列化的文档注解

javadoc 提供了文档注解(`@serial`,`@serialField`和`@serialData`)来标注序列化的相关信息。

* `@serial` 用于标注被序列化的字段, 注解后面可以加上对字段的描述,例如:`@serial FIELD_DESC`
* `@serialField` 用于标注serialPersistentFields 数组中的一个ObjectStreamField,用法:c` @serialField field-name field-type field-description`
* `@serialData` 用于标注被`writeObject`或`Externalizable.writeExternal`方法序列化的数据的序列和类型，用法:`@serialData data-description`

## ObjectStreamClass 

ObjectStreamClass 是存储在序列化流中的类的描述器,提供了序列化类的信息，例如class的全名，序列化版本UID，和类的字段信息。

ObjectStreamClass描述符还用于提供有关保存在序列化流中的动态代理类（例如，通过调用java.lang.reflect.Proxy的getProxyClass方法获得的类）的信息。动态代理类本身没有可序列化的字段，并且serialVersionUID为0L。换句话说，将动态代理类的Class对象传递给ObjectStreamClass的静态查找方法时，返回的ObjectStreamClass实例将具有以下属性：

1. 调用其getSerialVersionUID方法将返回0L。
2. 调用其getFields方法将返回长度为零的数组。
3. 使用任何String参数调用其getField方法将返回null。

ObjectStreamClass实例的序列化形式取决于它表示的Class对象是可序列化的，可外部化的还是动态代理类。当将ObjectStreamClass不代表动态代理类的实例写入流中时，它将写入类名称和serialVersionUID，标志以及字段数。根据类的不同，可能会编写其他信息:

1. 对于不可序列化的类，字段数始终为零。标志位 `SC_SERIALIZABLE`,`SC_EXTERNALIZABLE`不会被设置
2. 对于可序列化的类，标志位`SC_SERIALIZABLE`会被设置，字段数是可序列化字段的数，后面是每个可序列化字段的描述符。描述符以一定规范顺序编写。首先写入按字段名称排序原始类型字段的描述符，然后是对象类型字段的描述符（按字段名称排序）。名称使用String.compareTo进行排序。
3. 对于可外部化的类，标志SC_EXTERNALIZABLE，并且字段数始终为零。
4. 对于枚举类型，标志SC_ENUM，并且字段数始终为零。
5. 当ObjectOutputStream序列化动态代理类的ObjectStreamClass描述符（通过将其Class对象传递给java.lang.reflect.Proxy的isProxyClass方法确定）时，它将写入动态代理类实现的接口数，然后是接口名称。通过在动态代理类的Class对象上调用getInterfaces方法以返回接口的顺序列出它们。动态代理类和非动态代理类的ObjectStreamClass描述符的序列化表示形式通过使用不同的类型代码（分别为TC_PROXYCLASSDESC和TC_CLASSDESC）来区分。

## 流唯一标识符(serialVersionUID )

默认流唯一标识符是类名称，接口类名称，方法和字段的64位哈希值。如果未为某个类声明SUID，则该值默认为该类的哈希。动态代理类和枚举类型总是有值0L。

强烈建议所有可序列化的类显式声明 serialVersionUID值，因为默认 serialVersionUID计算对类详细信息高度敏感，而类详细信息可能会根据编译器的实现而变化，因此可能在serialVersionUID 反序列化期间导致意外冲突，从而导致反序列化失败。

## 二进制协议

语法文件如下:

```BNF
stream:
  magic version contents

contents:
  content
  contents content

content:
  object
  blockdata

object:
  newObject
  newClass
  newArray
  newString
  newEnum
  newClassDesc
  prevObject
  nullReference
  exception
  TC_RESET

newClass:
  TC_CLASS classDesc newHandle

classDesc:
  newClassDesc
  nullReference
  (ClassDesc)prevObject      // an object required to be of type
                             // ClassDesc

superClassDesc:
  classDesc

newClassDesc:
  TC_CLASSDESC className serialVersionUID newHandle classDescInfo
  TC_PROXYCLASSDESC newHandle proxyClassDescInfo
classDescInfo:
  classDescFlags fields classAnnotation superClassDesc 

className:
  (utf)

serialVersionUID:
  (long)

classDescFlags:
  (byte)                  // Defined in Terminal Symbols and
                            // Constants

proxyClassDescInfo:
  (int)<count> proxyInterfaceName[count] classAnnotation
      superClassDesc
proxyInterfaceName:
  (utf)
fields:
  (short)<count>  fieldDesc[count]

fieldDesc:
  primitiveDesc
  objectDesc

primitiveDesc:
  prim_typecode fieldName

objectDesc:
  obj_typecode fieldName className1

fieldName:
  (utf)

className1:
  (String)object             // String containing the field's type,
                             // in field descriptor format
classAnnotation:
  endBlockData
  contents endBlockData      // contents written by annotateClass

prim_typecode:
  `B'       // byte
  `C'       // char
  `D'       // double
  `F'       // float
  `I'       // integer
  `J'       // long
  `S'       // short
  `Z'       // boolean

obj_typecode:
  `[`   // array
  `L'       // object

newArray:
  TC_ARRAY classDesc newHandle (int)<size> values[size]

newObject:
  TC_OBJECT classDesc newHandle classdata[]  // data for each class

classdata:
  nowrclass                 // SC_SERIALIZABLE & classDescFlag &&
                            // !(SC_WRITE_METHOD & classDescFlags)
  wrclass objectAnnotation  // SC_SERIALIZABLE & classDescFlag &&
                            // SC_WRITE_METHOD & classDescFlags
  externalContents          // SC_EXTERNALIZABLE & classDescFlag &&
                            // !(SC_BLOCKDATA  & classDescFlags
  objectAnnotation          // SC_EXTERNALIZABLE & classDescFlag&& 
                            // SC_BLOCKDATA & classDescFlags

nowrclass:
  values                    // fields in order of class descriptor

wrclass:
  nowrclass

objectAnnotation:
  endBlockData
  contents endBlockData     // contents written by writeObject
                            // or writeExternal PROTOCOL_VERSION_2.

blockdata:
  blockdatashort
  blockdatalong

blockdatashort:
  TC_BLOCKDATA (unsigned byte)<size> (byte)[size]

blockdatalong:
  TC_BLOCKDATALONG (int)<size> (byte)[size]

endBlockData   :
  TC_ENDBLOCKDATA

externalContent:          // Only parseable by readExternal
  ( bytes)                // primitive data
    object

externalContents:         // externalContent written by 
  externalContent         // writeExternal in PROTOCOL_VERSION_1.
  externalContents externalContent

newString:
  TC_STRING newHandle (utf)
  TC_LONGSTRING newHandle (long-utf)

newEnum:
  TC_ENUM classDesc newHandle enumConstantName
enumConstantName:
  (String)object
prevObject
  TC_REFERENCE (int)handle

nullReference
  TC_NULL

exception:
  TC_EXCEPTION reset (Throwable)object         reset 

magic:
  STREAM_MAGIC

version
  STREAM_VERSION

values:          // The size and types are described by the
                 // classDesc for the current object

newHandle:       // The next number in sequence is assigned
                 // to the object being serialized or deserialized

reset:           // The set of known objects is discarded
                 // so the objects of the exception do not
                 // overlap with the previously sent objects 
                 // or with objects that may be sent after 
                 // the exception
```


## 参考

- [1] [oracle java 8 serialization](https://docs.oracle.com/javase/8/docs/platform/serialization/spec/serialTOC.html)

