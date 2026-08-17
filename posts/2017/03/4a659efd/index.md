# Java克隆


Java的所有类都默认继承`java.lang.Object`类，在java.lang.Object类中有一个方法`clone()`。JDK API的说明文档解释这个方法将返回Object对象的一个拷贝。要说明的有两点：

1. 拷贝对象返回的是一个新对象，而不是一个引用。
2. 拷贝对象与用 new操作符返回的新对象的区别就是这个拷贝已经包含了一些原来对象的信息，而不是对象的初始信息。

<!--more-->

## `clone()`的实现

```java
public class CloneClass implements Cloneable {
    public int aInt;

    public Object clone() {
        CloneClass o = null;
        try {
            // 调用父类的super.clone方法（父类必须实现clone方法）
            //这个方法将最终调用Object的中native型的clone方法完成浅拷贝
            o = (CloneClass) super.clone(); 
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return o;
    }
}
```

实现`Cloneable`接口,Cloneable接口是不包含任何方法的！其实这个接口仅仅是一个标志，而且这个标志也仅仅是针对 Object类中`clone()`方法的，如果clone类没有实现Cloneable接口，并调用了Object的`clone()`方法\(也就是调用了 super.Clone()方法\)，那么Object的`clone()`方法就会抛出`CloneNotSupportedException`异常。

## `clone()`和`new`的区别

在java语言中，有2种方式可以创建对象：

1. 使用new操作符创建一个对象
2. 使用clone方法复制一个对象

*那么这两种方式有什么相同和不同呢？*

1. new操作符的本意是分配内存。程序执行到new操作符时， 首先去看new操作符后面的类型，因为知道了类型，才能知道要分配多大的内存空间。分配完内存之后，再调用构造函数，填充对象的各个域，这一步叫做对象的初始化，构造方法返回后，一个对象创建完毕，可以把他的引用（地址）发布到外部，在外部就可以使用这个引用操纵这个对象。
2. 而clone在第一步是和new相似的， 都是分配内存，调用clone方法时，分配的内存和源对象（即调用clone方法的对象）相同，然后再使用原对象中对应的各个域，填充新对象的域， 填充完成之后，clone方法返回，一个新的相同的对象被创建，同样可以把这个新对象的引用发布到外部。

> 注意：clone方法是从内存中拷贝，重新分配新内存，这个过程中新对象没有执行构造器方法。

## 上例子

* `new`

```java
Person p = new Person(23, "zhang");  
Person p1 = p;  
  
System.out.println(p);  
System.out.println(p1); 
/* output
 *----------------------
 * Person@2f9ee1ac
 * Person@2f9ee1ac
 */
```

* `clone()`

```java
Person p = new Person(23, "zhang");  
Person p1 = (Person) p.clone();  
  
System.out.println(p);  
System.out.println(p1); 
/* output
 *----------------------
 * Person@2f9ee1ac
 * Person@67f1fba0
 */ 
```

显然，`=` 操作只是把句柄的引用地址传递出去了，所以句柄指向同一个地址。而`clone()`创建了一个新的对象。

## 影子clone和深度clone

```java
public class Person implements Cloneable{  
      
    private int age ;  
    private List<String> names;
    
      
    public Person(int age, List<String> names) {  
        this.age = age;  
        this.names = names;  
    }  
      
    public Person() {}  
  
    public int getAge() {  
        return age;  
    }  
  
    public List<String> getNames() {  
        return names;  
    }  
      
    @Override  
    protected Object clone() throws CloneNotSupportedException {  
        return (Person)super.clone();  
    }  
}
```

Person中有两个成员变量，分别是name和age， names是`List<String>`类型， age是int类型。

names是`List<String>`类型的，它只是一个引用，指向一个真正的`List<String>`对象，那么对它的拷贝有两种方式：直接将源对象中的names的引用值拷贝给新对象的names字段，或者是根据原Person对象中的names指向的对象创建一个新的相同的对象，将这个新对象的引用赋给新拷贝的Person对象的names字段。这两种拷贝方式分别叫做影子克隆和深度克隆。默认的克隆方式是影子克隆，实现深度克隆可以通过重写clone方法，除了调用父类中的clone方法得到新的对象，还要将该类中的引用变量也clone出来。

```java
    @Override  
    protected Object clone() throws CloneNotSupportedException {  
        Person person =  (Person)super.clone();  
        person.names = (List<String>)this.names.clone();// names 不能为 final
        return person;
    } 
```


* 由于java5.0后引入了协变返回类型（covariant return type）实现（基于泛型），即覆盖方法的返回类型可以是被覆盖方法的返回类型的子类型，所以clone方法可以直接返回实际类型，而不用返回Object类型，然后客户端再强转。
* 要实现深clone，引用属性不能为final。因为super.clone()时已经给final field 赋一次值了，不能再去修正克隆对象的final field。
* 数组的clone，仅仅复制的是数组中的元素，即若数组中元素为引用类型，仅仅复制引用。System.arraycopy也是重新开辟了空间，复制了数组中的元素

