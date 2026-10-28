# 设计模式之单例模式

单例模式(Singleton Pattern)：确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例，这个类称为单例类，它提供全局访问的方法。单例模式是一种对象创建型模式。

<!--more-->

## 基于类加载机制的饿汉式

### 代码实现

```java
class EagerSingleton {   
    private static final EagerSingleton instance = new EagerSingleton();   
    private EagerSingleton() { }   
  
    public static EagerSingleton getInstance() {  
        return instance;   
    }     
} 
```

## 基于类加载机制的静态内部类实现单例

```java
class Singleton {  
    private Singleton() {  
    }  
    private static class HolderClass {  
            private final static Singleton instance = new Singleton();  
    }  
    public static Singleton getInstance() {  
        return HolderClass.instance;  
    }
}
```

第一次调用`getInstance()`时将加载内部类`HolderClass`，在该内部类中定义了一个`static`类型的变量`instance`，此时会首先初始化这个成员变量，由Java虚拟机来保证其线程安全性，确保该成员变量只能初始化一次。由于`getInstance()`方法没有任何线程锁定，因此其性能不会造成任何影响。

## 懒汉式

```java
class LazySingleton {   
    private static LazySingleton instance = null;   
  
    private LazySingleton() { }   
  
    synchronized public static LazySingleton getInstance() {   
        if (instance == null) {  
            instance = new LazySingleton();   
        }  
        return instance;   
    }  
}  
```

懒汉式单例在第一次调用`getInstance()`方法时实例化，在类加载时并不自行实例化，这种技术又称为延迟加载`(Lazy Load)`技术，即需要的时候再加载实例。

### 线程安全的懒汉式

为了避免多个线程同时调用`getInstance()`方法，我们可以使用关键字`synchronized`，代码如下所示：

```java
class LazySingleton {   
    private static LazySingleton instance = null;   
  
    private LazySingleton() { }   
  
    synchronized public static LazySingleton getInstance() {   
        if (instance == null) {  
            instance = new LazySingleton();   
        }  
        return instance;   
    }  
}  
```

### 节省资源的双重校验锁

懒汉式单例类在`getInstance()`方法前面增加了关键字`synchronized`进行线程锁，以处理多个线程同时访问的问题。但是，上述代码虽然解决了线程安全问题，但是每次调用`getInstance()`时都需要进行线程锁定判断，在多线程高并发访问环境中，将会导致系统性能大大降低。如何既解决线程安全问题又不影响系统性能呢？我们继续对懒汉式单例进行改进。事实上，我们无须对整个`getInstance()`方法进行锁定，只需对其中的代码`instance = new LazySingleton();`进行锁定即可。因此getInstance()方法可以进行如下改进：

```java
public static LazySingleton getInstance() {   
    if (instance == null) {  
        synchronized (LazySingleton.class) {  
            instance = new LazySingleton();   
        }  
    }  
    return instance;   
}  
```

__注意__，新的问题出现了。假如在某一瞬间线程A和线程B都在调用`getInstance()`方法，此时`instance`对象为`null`值，均能通过`instance == null`的判断。由于实现了`synchronized`加锁机制，线程A进入`synchronized`锁定的代码中执行实例创建代码，线程B处于排队等待状态，必须等待线程A执行完毕后才可以进入`synchronized`锁定代码。但当A执行完毕时，线程B并不知道实例已经创建，将继续创建新的实例，导致产生多个单例对象，违背单例模式的设计思想，因此需要进行进一步改进，在`synchronized`中再进行一次`(instance == null)`判断，这种方式称为双重检查锁定(Double-Check Locking)。使用双重检查锁定实现的懒汉式单例类完整代码如下所示：

```java
class LazySingleton {   
    private volatile static LazySingleton instance = null;   
    private LazySingleton() { }   
    public static LazySingleton getInstance() {   
        //第一重判断  
        if (instance == null) {  
            //锁定代码块  
            synchronized (LazySingleton.class) {  
                //第二重判断  
                if (instance == null) {  
                    instance = new LazySingleton(); //创建单例实例  
                }  
            }  
        }  
        return instance;   
    }  
}
```

 > 值得一提的是此处的单例对象 instance 必须使用`volatile`修饰。声明成 volatile 的变量的指令不会被重新排序。

### 关于双重校验锁的分析

```java
public static Singleton getInstance()
{
 if (instance == null)
  {
      synchronized(Singleton.class) {  //1
      if (instance == null)          //2
        instance = new Singleton();  //3
    }
  }
  return instance;
}
```

> 双重检查锁定背后的理论是：在 //2 处的第二次检查使创建两个不同的 Singleton 对象成为不可能。

假设有下列事件序列：

1. 线程 1 进入 getInstance() 方法。
2. 由于 instance 为 null ，线程 1 在 //1 处进入 synchronized 块。
3. 线程 1 被线程 2 预占。
4. 线程 2 进入 getInstance() 方法。
5. 由于 instance 仍旧为 null ，线程 2 试图获取 //1 处的锁。然而，由于线程 1 持有该锁，线程 2 在 //1 处阻塞。
6. 线程 2 被线程 1 预占。
7. 线程 1 执行，由于在 //2 处实例仍旧为 null ，线程 1 还创建一个 Singleton 对象并将其引用赋值给 instance 。
8. 线程 1 退出 synchronized 块并从 getInstance() 方法返回实例。
9. 线程 1 被线程 2 预占。
10. 线程 2 获取 //1 处的锁并检查 instance 是否为 null 。
11. 由于 instance 是非 null 的，并没有创建第二个 Singleton 对象，由线程 1 创建的对象被返回。

双重检查锁定背后的理论是完美的。不幸地是，现实完全不同。双重检查锁定的问题是：并不能保证它会在单处理器或多处理器计算机上顺利运行。双重检查锁定失败的问题并不归咎于 JVM 中的实现 bug，而是归咎于 Java 平台内存模型。内存模型允许所谓的“重排序”，是失败的一个主要原因。

假设代码执行以下事件序列：

1. 线程 1 进入 getInstance() 方法。
2. 由于 instance 为 null ，线程 1 在 //1 处进入 synchronized 块。
3. 线程 1 前进到 //3 处，但在构造函数执行之前 ，使实例成为非 null 。
4. 线程 1 被线程 2 预占。
5. 线程 2 检查实例是否为 null 。因为实例不为 null，线程 2 将 instance 引用返回给一个构造完整但部分初始化了的 Singleton 对象。
6. 线程 2 被线程 1 预占。
7. 线程 1 通过运行 Singleton 对象的构造函数并将引用返回给它，来完成对该对象的初始化。

此事件序列发生在线程 2 返回一个尚未执行构造函数的对象的时候。为展示此事件的发生情况，假设为代码行 ·instance =new Singleton();`执行了下列伪代码：

```java
mem = allocate();             //Allocate memory for Singleton object.
instance = mem;               //Note that instance is now non-null, but has not been initialized.
ctorSingleton(instance);      //Invoke constructor for Singleton passing instance.
```

> 解决上述问题的方法是使用volatile关键字，在Java 1.5后volatile有了禁止指令重排序优化的功能

## 基于枚举的单例

```java
public class EnumSingleton{
    private EnumSingleton(){}
    public static EnumSingleton getInstance(){
        return Singleton.INSTANCE.getInstance();
    }
    private static enum Singleton{
        INSTANCE;
        
        private EnumSingleton singleton;
        //JVM会保证此方法绝对只调用一次
        private Singleton(){
            singleton = new EnumSingleton();
        }
        public EnumSingleton getInstance(){
            return singleton;
        }
    }
}
```

在枚举中我们明确了构造方法限制为私有，在我们访问枚举实例时会执行构造方法，同时每个枚举实例都是static final类型的，也就表明只能被实例化一次。在调用构造方法时，我们的单例被实例化。 
也就是说，因为enum中的实例被保证只会被实例化一次，所以我们的INSTANCE也被保证实例化一次。

### 枚举的优势

上面4种单例方法还存在问题:

* 序列化可能会破坏单例模式，比较每次反序列化一个序列化的对象实例时都会创建一个新的实例,而enum 无法序列化和反序列化;

```java
public class Singleton implements java.io.Serializable {
   public static Singleton INSTANCE = new Singleton();

   protected Singleton() {
   }

   //反序列时直接返回当前INSTANCE
   private Object readResolve() {
            return INSTANCE;
      }
}
```

* 使用反射强行调用私有构造器，解决方式可以修改构造器，让它在创建第二个实例的时候抛异常;

```java
public static Singleton INSTANCE = new Singleton();
private static volatile  boolean  flag = true;
private Singleton(){
    if(flag){
    flag = false;
    }else{
        throw new RuntimeException("The instance  already exists ！");
    }
}
```

* 支持clone的类，使用clone 方法会创建一个新实例，而enum 无法clone;

使用Enum可以很好的避免上面问题：

#### 序列化

枚举序列化是由jvm保证的，每一个枚举类型和定义的枚举变量在JVM中都是唯一的，在枚举类型的序列化和反序列化上，Java做了特殊的规定：在序列化时Java仅仅是将枚举对象的name属性输出到结果中，反序列化的时候则是通过java.lang.Enum的valueOf方法来根据名字查找枚举对象。同时，编译器是不允许任何对这种序列化机制的定制的并禁用了writeObject、readObject、readObjectNoData、writeReplace和readResolve等方法，从而保证了枚举实例的唯一性。

```java
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
```

实际上通过调用enumType(Class对象的引用)的enumConstantDirectory方法获取到的是一个Map集合，在该集合中存放了以枚举name为key和以枚举实例变量为value的Key&Value数据，因此通过name的值就可以获取到枚举实例，看看enumConstantDirectory方法源码：

```java
Map<String, T> enumConstantDirectory() {
        if (enumConstantDirectory == null) {
            //getEnumConstantsShared最终通过反射调用枚举类的values方法
            T[] universe = getEnumConstantsShared();
            if (universe == null)
                throw new IllegalArgumentException(
                    getName() + " is not an enum type");
            Map<String, T> m = new HashMap<>(2 * universe.length);
            //map存放了当前enum类的所有枚举实例变量，以name为key值
            for (T constant : universe)
                m.put(((Enum<?>)constant).name(), constant);
            enumConstantDirectory = m;
        }
        return enumConstantDirectory;
    }
    private volatile transient Map<String, T> enumConstantDirectory = null;
```

#### 反射

再来看看反射到底能不能创建枚举，下面试图通过反射获取构造器并创建枚举:

```java
public static void main(String[] args) throws IllegalAccessException, InvocationTargetException, InstantiationException, NoSuchMethodException {
  //获取枚举类的构造函数
   Constructor<SingletonEnum> constructor=SingletonEnum.class.getDeclaredConstructor(String.class,int.class);
   constructor.setAccessible(true);
   //创建枚举
   SingletonEnum singleton=constructor.newInstance("otherInstance",9);
  }
```

执行报错:

```java
Exception in thread "main" java.lang.IllegalArgumentException: Cannot reflectively create enum objects
    at java.lang.reflect.Constructor.newInstance(Constructor.java:417)
    at zejian.SingletonEnum.main(SingletonEnum.java:38)
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
    at java.lang.reflect.Method.invoke(Method.java:498)
    ...
```

看看newInstance方法源码:

```java
public T newInstance(Object ... initargs)
        throws InstantiationException, IllegalAccessException,
               IllegalArgumentException, InvocationTargetException
    {
        if (!override) {
            if (!Reflection.quickCheckMemberAccess(clazz, modifiers)) {
                Class<?> caller = Reflection.getCallerClass();
                checkAccess(caller, clazz, null, modifiers);
            }
        }
        //这里判断Modifier.ENUM是不是枚举修饰符，如果是就抛异常
        if ((clazz.getModifiers() & Modifier.ENUM) != 0)
            throw new IllegalArgumentException("Cannot reflectively create enum objects");
        ConstructorAccessor ca = constructorAccessor;   // read volatile
        if (ca == null) {
            ca = acquireConstructorAccessor();
        }
        @SuppressWarnings("unchecked")
        T inst = (T) ca.newInstance(initargs);
        return inst;
    }
```

## 单例模式的应用

### 优点

* 内存中只有一个实例，减少了内存开销，特别是当一个对象需要频繁地创建、销毁时，单例有很大优势。
* 避免对资源的多重占用，以读写文件为例，单例模式可以避免对同一资源文件的同时写操作。
* 在系统设置全局的访问点、优化和共享资源访问。

### 缺点

* 单例模式一般没有接口，不便扩展。
* 与单一职责原则有冲突，一个类是否单例应该取决于环境，而不是使用单例模式。

### 使用场景

* 要求一个类有且只有一个对象，那么可以考虑单例模式。
* 创建对像需要消耗资源过多。
* 需要定义大量的静态常量和静态方法的环境\(如工具类\)

