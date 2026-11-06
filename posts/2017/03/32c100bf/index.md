# Java自增自减-中间变量缓存


本文介绍了 Java 自增/减操作符的中间变量缓存机制。

<!--more-->

```java
public class Demo {
    public static  void main(String[] args){
        method_1();
        method_2();
    }
    private static void method_1(){
        int j = 0;
        for(int i=0;i<100;i++){
            j = j++;
        }
        System.out.println("method_1---"+j);
    }
    private static void method_2(){
        int j = 0;
        for(int i=0;i<100;i++){
            j = ++j;
        }
        System.out.println("method_2---"+j);
    }
}
```

上面程序的输出结果是：

```java
method_1---0  
method_2---100 
```

为什么会是这样呢？

因为在计算过程中，使用了Java中间变量缓存机制。在java中，执行自增运算时，会为每一个自增操作分配一个临时变量。

- 如果是前缀加`++i`，就会先自加1后赋值给临时变量；

```java
j = j++;
// 转化为下面代码
{
  int temp = j; 
  j = j + 1; 
  j = temp;
}
```

- 如果是后缀加`i++`，就会先赋值给临时变量后自加1。

```java
j = ++j;
// 转化为下面代码
{
  j = j + 1;
  int temp = j;
  j = temp; 
}
```
运算最终使用的，并不是变量本身，而是被赋了值的临时变量。

