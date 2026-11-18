# Java，循环还是递归？

开始比较前，先来了解下循环和递归的定义。

- 循环是一段在程序中只出现一次，但可能会连续执行多次的代码。循环中的代码会执行特定的次数(例如for)，或者是执行到特定条件成立时结束循环(例如while)，或者是针对某一集合中的所有项目都执行一次。
- 递归指在函数的定义中使用函数自身的方法，即一个函数不断调用自身的行为。

>注意: 上面的循环定义不适用于所有情况，在一些函数编程语言（例如Haskell和Scheme）中会使用递归或不动点组合子来达到循环的效果。

<!--more-->

## 一个经典例子-斐波那契数列

递归方式:

```java
// 递归方式
int Fibonacci(int n)
{
    if(n<= 0){
        return 0;
    }
    if(n == 1){
        return 1;
    }
    return Fibonacci(n-1)+Fibonacci(n-2);
}
```

循环方式:

```java
int Fibonacci(int n)
{
    int res[2] = { 0, 1 };
    if (n < 2){
    return res[n];
    }
    int pre_pre = 1;
    int pre = 0;
    int result = 0;
    for (int i = 2; i <= n; i++)
    {
        result = pre + pre_pre;
        pre_pre = pre;
        pre = result;
    }
    return result;
}
```

对比代码会发现，递归显得更整洁，函数里面调用函数自身，循环代码显得比较多

但是递归在性能上是不如循环的。函数调用有时间和空间的额外消耗(例如:C语言[<sup>1</sup>](#refer-anchor-1)，Java语言[<sup>2</sup>](#refer-anchor-2))：

- 每一次函数调用，都需要在内存栈中分配空间以保存参数、返回地址及临时变量，这些空间的占用只有在递归退出时才会释放。
- 当递归调用的层级太多的时候，就会超出栈的容量，从而导致调用栈溢出。而且往栈里压入数据和弹出数据都需要时间。

## 尾递归优化

尾调用是指一个函数里的最后一个动作是返回一个函数的调用结果的情形，即最后一步新调用的返回值直接被当前函数的返回结果。该尾部调用位置被称为尾位置。尾调用原则上都可以通过简化函数调用栈的结构而获得性能优化（称为`尾调用消除`），但是优化尾调用是否方便可行取决于运行环境对此类优化的支持程度如何。

尾调用可能位于一个函数语法上最后的位置：

```
function foo(data) {
    a(data);
    return b(data);
}
```

但是并非所有的尾调用都必须在一个函数语法上最后的位置。在下面的例子中，b、c 的调用都在尾位置。

```
function bar(data) {
    if ( a(data) ) {
        return b(data);
    }
    return c(data);
}
```

若函数在尾位置调用自身（或是一个尾调用本身的其他函数等等），则称这种情况为尾递归。尾递归也是递归的一种特殊情形。尾递归是一种特殊的尾调用，即在尾部直接调用自身的递归函数。尾递归在普通尾调用的基础上，多出了2个特征：

- 在尾部调用的是函数自身 （Self-called）；
- 可通过优化，使得计算仅占用常量栈空间 (Stack Space)。


假设有以下递归函数:

```python
def recsum(x):
  if x == 1:
    return x
  else:
    return x + recsum(x - 1)
```

调用`recsum(5)`的栈空间变化

```
recsum(5)
5 + recsum(4)
5 + (4 + recsum(3))
5 + (4 + (3 + recsum(2)))
5 + (4 + (3 + (2 + recsum(1))))
5 + (4 + (3 + (2 + 1)))
5 + (4 + (3 + 3))
5 + (4 + 6)
5 + 10
15
```

如果将上面的函数修改为尾递归形式:

```python
def tailrecsum(x, running_total=0):
  if x == 0:
    return running_total
  else:
    return tailrecsum(x - 1, running_total + x)
```

尾递归的栈空间变化:

```
tailrecsum(5, 0) 
tailrecsum(4, 5) 
tailrecsum(3, 9)
tailrecsum(2, 12) 
tailrecsum(1, 14) 
tailrecsum(0, 15) 
15
```

可见在经过尾递归优化后，等效为:

```python
for i in range(6):
  sum += i
```

但是不幸的是Java不支持尾递归优化。

## Java 如何支持大深度递归

治标不治本的方法: 通过`-Xss4M`调整每个线程的堆栈大小。

此外还可以借鉴函数式编程语言的思路，通过函数式方法模拟递归。

```java
/**
 * 尾递归
 * @param <T> 迭代参数类型
 * @param <R> 迭代结果
 */
@FunctionalInterface
public interface TailRecursion<T,R> {
    /**
     * 函数实际逻辑。
     * func(T,R) -> func(T',R')
     * @return func
     */
    TailRecursion<T,R> apply();

    default boolean isFinished(){
        return false;
    }
    default R getResult() {
        throw new IllegalStateException("Recursion not End");
    }

    @SuppressWarnings("OptionalGetWithoutIsPresent")
    default R invoke() {
        // 此处通过stream模拟循环执行。
        return Stream.iterate(this, TailRecursion::apply)
                .filter(TailRecursion::isFinished)
                .findFirst().get()
                .getResult();
    }

    static < T,R> TailRecursion<T, R> done (final R value) {
        return new TailRecursion <T,R>() {
            @Override
            public boolean isFinished () { return true ; }
            @Override
            public R getResult () { return value ; }
            @Override
            public TailRecursion<T,R> apply () {
                throw new IllegalStateException ( "Recursion Ended" );
            }
        };
    }
}
```

基于 TailRecursion，我们可以实现斐波那契数列:

```java
public class Fibonacci
{
    public static class FibonacciResult{
        public Long pre;
        public Long current;

        public Integer index;

        public FibonacciResult(Long pre, Long current,Integer index)
        {
            this.pre = pre;
            this.current = current;
            this.index = index;
        }

        public static FibonacciResult of(Long pre,Long current,Integer index){
            return new FibonacciResult(pre, current,index);
        }
    }
    public static TailRecursion<Integer,FibonacciResult> fibonacci(Integer n,FibonacciResult r){
        if(n ==0){
            return TailRecursion.done(FibonacciResult.of(-1L,0L,0));
        }
        else if (n ==1) {
            return TailRecursion.done(FibonacciResult.of(0L,1L,1));
        }else {
            if(n>r.index){
                return ()->fibonacci(n, FibonacciResult.of(r.current, r.pre +r.current,++r.index));
            }else {
                // n=r.index
                return TailRecursion.done(FibonacciResult.of(r.pre,r.current,r.index));
            }
        }
    }

    public static Long fibonacci(Integer n){
        return fibonacci(n,FibonacciResult.of(0L,1L,1)).invoke().current;
    }

    public static void main(String[] args)
    {
        System.out.println(fibonacci(8));
    }
}
```

## 总结

- 循环:
    - 优点： 结构简单
    - 缺点： 并不能解决所有的问题。有的问题适合使用递归而不是循环，如果使用循环并不困难的话，最好使用循环。
- 递归
    - 优点： 代码简洁、清晰，并且容易验证正确性
    - 缺点： 它的运行需要较多次数的方法调用，如果调用层数比较深，需要增加额外的堆栈处理，比如参数传递需要压栈等操作，会对执行效率有一定影响。

循环次数不是特别大，处理逻辑及其复杂，如果用循环算法，可能难于理解时，可优先采用递归算法。处理逻辑简单，则尽量用循环。

## 参考

<div id="refer-anchor-1"></div>

- [1] [Memory of C-栈是如何工作的](/posts/26272894/#栈如何工作)

<div id="refer-anchor-2"></div>

- [2] [Java虚拟机-虚拟机栈](/posts/a438de3b/)

