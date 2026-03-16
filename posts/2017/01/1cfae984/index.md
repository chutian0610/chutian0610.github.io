# 设计模式之迭代器模式

在软件开发中，我们经常需要使用聚合对象来存储一系列数据。聚合对象拥有两个职责：一是存储数据；二是遍历数据。从依赖性来看，前者是聚合对象的基本职责；而后者既是可变化的，又是可分离的。因此，可以将遍历数据的行为从聚合对象中分离出来，封装在一个被称之为“迭代器”的对象中，由迭代器来提供遍历聚合对象内部数据的行为，这将简化聚合对象的设计，更符合“单一职责原则”的要求。

<!--more-->

## 迭代器模式定义

迭代器模式定义如下：迭代器模式(Iterator Pattern)：提供一种方法来访问聚合对象，而不用暴露这个对象的内部表示，其别名为游标(Cursor)。迭代器模式是一种对象行为型模式。在迭代器模式结构中包含聚合和迭代器两个层次结构，考虑到系统的灵活性和可扩展性，在迭代器模式中应用了工厂方法模式，其模式结构如图所示：

```mermaid
classDiagram
class Aggragate{
    +iterator() int
}
&lt;&lt;interface>> Aggragate
class ConcreteAggregate{
    +iterator() int
}
Aggragate <|.. ConcreteAggregate
class Iterator{
    +first() Aggragate
    +next() Aggragate
    +hasNext() Aggragate
    +currentItem() Aggragate
}
&lt;&lt;interface>> Iterator
class ConcreteIterator{
    -Aggragate aggragate
    +first() Aggragate
    +next() Aggragate
    +hasNext() Aggragate
    +currentItem() Aggragate
}
Iterator <|.. ConcreteIterator
Aggragate --*ConcreteIterator
```

在迭代器模式结构图中包含如下几个角色：

* Iterator（抽象迭代器）：它定义了访问和遍历元素的接口，声明了用于遍历数据元素的方法，例如：用于获取第一个元素的first()方法，用于访问下一个元素的next()方法，用于判断是否还有下一个元素的hasNext()方法，用于获取当前元素的currentItem()方法等，在具体迭代器中将实现这些方法。
* ConcreteIterator（具体迭代器）：它实现了抽象迭代器接口，完成对聚合对象的遍历，同时在具体迭代器中通过游标来记录在聚合对象中所处的当前位置，在具体实现时，游标通常是一个表示位置的非负整数。
* Aggregate（抽象聚合类）：它用于存储和管理元素对象，声明一个createIterator()方法用于创建一个迭代器对象，充当抽象迭代器工厂角色。
* ConcreteAggregate（具体聚合类）：它实现了在抽象聚合类中声明的createIterator()方法，该方法返回一个与该具体聚合类对应的具体迭代器ConcreteIterator实例。

## 代码示例

```java
// 在抽象迭代器中声明了用于遍历聚合对象中所存储元素的方法
interface Iterator {  
    public Aggregate first(); //将游标指向第一个元素  
    public Aggregate next(); //将游标指向下一个元素  
    public boolean hasNext(); //判断是否存在下一个元素  
    public Aggregate currentItem(); //获取游标指向的当前元素  
}  
// 在具体迭代器中将实现抽象迭代器声明的遍历数据的方法
class ConcreteIterator implements Iterator {  
    private ConcreteAggregate objects; //维持一个对具体聚合对象的引用，以便于访问存储在聚合对象中的数据  
    private int cursor; //定义一个游标，用于记录当前访问位置  
    public ConcreteIterator(ConcreteAggregate objects) {  
        this.objects=objects;  
    }  
  
    public Aggregate first() {  ......  }  
          
    public Aggregate next() {  ......  }  
  
    public boolean hasNext() {  ......  }  
      
    public Aggregate currentItem() {  ......  }  
}  
// 聚合类用于存储数据并负责创建迭代器对象.
interface Aggregate {  
    Iterator iterator();  
} 
// 具体聚合类作为抽象聚合类的子类，一方面负责存储数据.
// 另一方面实现了在抽象聚合类中声明的工厂方法iterator()，用于返回一个与该具体聚合类对应的具体迭代器对象.
class ConcreteAggregate implements Aggregate {    
    ......    
    public Iterator iterator() {  
    return new iterator(this);  
    }  
    ......  
}  
```

## 迭代器模式总结

### 主要优点
 
迭代器模式的主要优点如下：

1. 它支持以不同的方式遍历一个聚合对象，在同一个聚合对象上可以定义多种遍历方式。在迭代器模式中只需要用一个不同的迭代器来替换原有迭代器即可改变遍历算法，我们也可以自己定义迭代器的子类以支持新的遍历方式。
2. 迭代器简化了聚合类。由于引入了迭代器，在原有的聚合对象中不需要再自行提供数据遍历等方法，这样可以简化聚合类的设计。
3. 在迭代器模式中，由于引入了抽象层，增加新的聚合类和迭代器类都很方便，无须修改原有代码，满足“开闭原则”的要求。
 
### 主要缺点

迭代器模式的主要缺点如下：
1. 由于迭代器模式将存储数据和遍历数据的职责分离，增加新的聚合类需要对应增加新的迭代器类，类的个数成对增加，这在一定程度上增加了系统的复杂性。
2. 抽象迭代器的设计难度较大，需要充分考虑到系统将来的扩展，例如JDK内置迭代器Iterator就无法实现逆向遍历，如果需要实现逆向遍历，只能通过其子类ListIterator等来实现，而ListIterator迭代器无法用于操作Set类型的聚合对象。在自定义迭代器时，创建一个考虑全面的抽象迭代器并不是件很容易的事情。
 
### 适用场景

在以下情况下可以考虑使用迭代器模式：

1. 访问一个聚合对象的内容而无须暴露它的内部表示。将聚合对象的访问与内部数据的存储分离，使得访问聚合对象时无须了解其内部实现细节。
2. 需要为一个聚合对象提供多种遍历方式。
3. 为遍历不同的聚合结构提供一个统一的接口，在该接口的实现类中为不同的聚合结构提供不同的遍历方式，而客户端可以一致性地操作该接口。

