# Java不可变对象

可变类和不可变类(Mutable and Immutable Objects)的初步定义：

1. 可变类：当你获得这个类的一个实例引用时，你可以改变这个实例的内容。 
2. 不可变类：当你获得这个类的一个实例引用时，你不可以改变这个实例的内容。不可变类的实例一但创建，其内在成员变量的值就不能被修改。

<!--more-->

JDK中的可变类和不可变类:

* 不可变类：`java.lang`包中 Boolean, Byte, Character, Double, Float, Integer, Long, Short,String;
* 可变类: StringBuffer,`java.util.Date`,常见的 POJO等。

## 如何创建一个不可变类

1. immutable对象的状态在创建之后就不能发生改变，尽量不要实现属性的setter方法，任何对它的改变都应该产生一个新的对象。
2. Immutable类的所有的属性都应该是final的，如需要实现getter，使用copy-on-write原则。
3. 对象必须被正确的创建，比如：对象引用在对象创建过程中不能泄露(leak)。
4. 对象应该是final的，以此来限制子类继承父类，以避免子类改变了父类的immutable特性^[1]。
5. 如果类中包含mutable类对象，那么返回给客户端的时候，返回该对象的一个拷贝，而不是该对象本身（该条可以归为第一条中的一个特例,同样，初始化时也应该传入一个拷贝)。

```java
public final class Contacts {

    private final String name;
    private final String mobile;

    public Contacts(String name, String mobile) {
        this.name = name;
        this.mobile = mobile;
    }
  
    public String getName(){
        return name;
    }
  
    public String getMobile(){
        return mobile;
    }
}
```

我们为类添加了final修饰，从而避免因为继承和多态引起的immutable风险。

上面是最简单的一种实现immutable类的方式，可以看到它的所有属性都是final的。

有时候你要实现的immutable类中可能包含mutable的类，比如java.util.Date,尽管你将其设置成了final的，但是它的值还是可以被修改的，为了避免这个问题，我们建议返回给用户该对象的一个拷贝，这也是Java的最佳实践之一。下面是一个创建包含mutable类对象的immutable类的例子：

```java
public final class ImmutableReminder{
    private final Date remindingDate;
  
    public ImmutableReminder (Date remindingDate) {
        if(remindingDate.getTime() < System.currentTimeMillis()){
            throw new IllegalArgumentException("Can not set reminder” +
                        “ for past time: " + remindingDate);
        }
 　　　　// this.remindingDate= remindingDate;     //error   
        this.remindingDate = new Date(remindingDate.getTime());
    } 
    public Date getRemindingDate() { 
        // return this.remindingDate;    //error
        return (Date) remindingDate.clone(); 
    } 
}
```

1. 不可变对象常用于解决并发问题，不可变对象一定是线程安全的。一旦线程试图修改对象，只会得到一个新的对象。不会影响原来的值。
2. 不可变对象的特性也符合事件驱动开发的范式，事件只能被创建和消费，修改可以通过transformer产生新的事件来实现。

[^1]: 使用final Class(强不可变类)，或者将所有类方法加上final(弱不可变类)

