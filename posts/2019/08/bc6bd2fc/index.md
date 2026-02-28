# functional programming 简介

函数式编程是一种编程范式,它把计算当成是数学函数的求值，从而避免改变状态和使用可变数据。它是一种声明式的编程范式，通过表达式和声明而不是语句来编程。函数式编程是幂等的(无状态的):函数的返回值仅取决于其参数，因此调用具有相同参数值的函数始终会产生相同的结果。这与命令式编程形成对比，在命令式编程中，除了函数的参数之外，程序状态可以影响函数的结果值。随着多核平台和并发计算的发展，函数式编程的无状态特性，在处理这些问题时有着其他编程范式不可比拟的天然优势。

<!--more-->

## 数学函数

数学中的函数是输入元素的集合到可能的输出元素的集合之间的映射关系，并且每个输入元素只能映射到一个输出元素。

用专业的术语来说，输入元素称为函数的参数（argument）。输出元素称为函数的值（value）。输入元素的集合称为函数的定义域（domain）。输出元素和其他附加元素的集合称为函数的到达域（codomain）。存在映射关系的输入和输出元素对的集合，称为函数的图形（graph）。输出元素的集合称为像（image）。这里需要注意像和到达域的区别。到达域还可能包含除了像中元素之外的其他元素，也就是没有输入元素与之对应的元素。

## λ 演算

函数式编程起源于 λ 演算。λ 演算实际上是对前面提到的函数概念的简化，方便以系统的方式来研究函数。λ 演算的函数有两个重要特征：

* λ 演算中的函数都是匿名的，没有显式的名称。比如函数 $sum(x, y) = x + y$ 可以写成 $(x, y) \to x + y$。由于函数本身仅由其映射关系来确定，函数名称实际上并没有意义。因此使用匿名函数是合理的。
* λ 演算中的函数都只有一个输入。有多个输入的函数可以转换成多个只包含一个输入的函数的嵌套调用。这个过程就是通常所说的柯里化（currying）。如 $(x, y) \to x + y$ 可以转换成 $x \to (y \to x + y)$。右边的函数的返回值是另外一个函数。这一限定简化了 λ 演算的定义。

λ 演算是基于 λ 项（λ-term）的语言。λ 项是 λ 演算的基本单元。λ 演算在 λ 项上定义了各种转换规则。λ 项由下面 3 个规则来定义:

```bnf
<expr> ::= <identifier>
<expr> ::= (λ <identifier> . <expr>)
<expr> ::= (<expr> <expr>)
```
语法规则定义的解释如下:

* 一个变量 x 就是一个 λ 项。
* 如果 M 是 λ 项，x 是一个变量，那么 (λx.M) 也是一个 λ 项。这样的 λ 项称为 λ 抽象（abstraction）。x 和 M 中间的点（.）用来分隔函数参数和内容。
* 如果M和N都是λ项，那么 (MN) 也是一个 λ 项。这样的λ项称为应用（application）。

所有的合法 λ 项，都只能通过重复应用上面的 3 个规则得来。需要注意的是，λ 项最外围的括号是可以省略的，也就是可以直接写为 λx.M 和 MN。当多个 λ 项连接在一起时，需要用括号来进行分隔，以确定 λ 项的解析顺序。

消歧约定:

1. 函数应用是左结合的，即：M N P意为(M N) P而非M (N P)。
2. 一个函数抽象的函数体将尽最大可能向右扩展，即：λx.M N代表的是一个函数抽象λx.(M N)而非函数应用(λx.M) N。在不出现歧义的情况下，可以省略括号。

### 绑定变量和自由变量

在 λ 抽象中，如果变量 x 出现在表达式中，那么该变量被绑定。表达式中绑定变量之外的其他变量称为自由变量。

举个例子:

* λx.xy：其中x是绑定变量，y是自由变量；
* (λy.y)(λx.xy)：这个表达式可以按括号划分为两个子表达式M和N，M的y是绑定变量，无自由变量，N的x是绑定变量，y是自由变量且与M无关；
* λx.(λy.xyz)：这个表达式中的x绑定于外部表达式，y绑定于内部表达式，z是自由变量。

一个λ演算表达式只有在其所有变量都是绑定的时候才完全合法。但是，当我们脱开上下文，关注于一个复杂表达式的子表达式时，自由变量是允许存在的,这时候搞清楚子表达式中的哪些变量是自由的就显得非常重要了。

### λ 演算运算法则

#### α 变换

α 变换（α-conversion）的目的是改变绑定变量的名称，避免名称冲突。比如，我们可以通过 α 变换把 λx.x+1 转换成 λy.y+1。如果两个λ项可以通过α变换来进行转换，则这两个 λ 项是 α 等价的。

对 λ 抽象进行 α 变换时，只能替换那些绑定到当前 λ 抽象上的变量。如 λ 抽象 λx.λx.x 可以 α 变换为 λx.λy.y 或 λy.λx.x，但是不能变换为 λy.λx.y。

#### β 约简

β 约简（β-reduction）与函数应用相关。在讨论 β 约简之前，需要先介绍替换的概念。对于 λ 项 M 来说，M[x := N] 表示把 λ 项 M 中变量 x 的自由出现替换成 N。具体的替换规则如下所示。A、B 和 M 是 λ 项，而 x 和 y 是变量。A ≡ B 表示两个 λ 项是相等的。

* x[x := M] ≡ M：直接替换一个变量 x 的结果是用来进行替换的 λ 项 M。
* y[x := M] ≡ y（x ≠ y）：y 是与 x 不同的变量，因此替换 x 并不会影响 y，替换结果仍然为 y。
* (AB)[x := M] ≡ (A[x := M]B[x := M])：A 和 B 都是 λ 项，(AB) 是 λ 项的应用。对 λ 项的应用进行替换，相当于替换之后再进行应用。
* (λx.A)[x := M] ≡ λx.A：这条规则针对 λ 抽象。如果 x 是 λ 抽象的绑定变量，那么不需要对 x 进行替换，得到的结果与之前的 λ 抽象相同。这是因为替换只是针对 M 中 x 的自由出现，如果 _x 在 M 中是不自由的，那么替换就不需要进行_ 。
* (λy.A)[x := M] ≡ λy.A[x := M]（x ≠ y 并且 y ∉ FV(M)）：这条规则也是针对λ抽象。λ 项 A 的绑定变量是 y，不同于要替换的 x，因此可以在 A 中进行替换动作。

在进行替换之前，可能需要先使用 α 变换来改变绑定变量的名称。比如，在进行替换 (λx.y)[y := x] 时，不能直接把出现的 y 替换成 x。这样就改变了之前的 λ 抽象的语义。正确的做法是先进行 α 变换，把 λx.y 替换成 λz.y，再进行替换，得到的结果是 λz.x。

替换的基本原则是要求在替换完成之后，原来的自由变量仍然是自由的。如果替换变量可能导致一个变量从自由变成绑定，需要首先进行 α 变换。在之前的例子中，λx.y 中的 x 是自由变量，而直接替换的结果 λx.x 把 x 变成了绑定变量，因此 α 变换是必须的。在正确的替换结果 λz.x 中，z 仍然是自由的。

#### η 变换

η 变换（η-conversion）描述函数的外延性（extensionality）。外延性指的是如果两个函数当且仅当对所有参数的结果相同时，才被认为是相等的。比如一个函数 F，当参数为 x 时，它的返回值是 Fx。那么考虑声明为 λy.Fy 的函数 G。函数 G 对于输入参数 x，同样返回结果 Fx。F 和 G 可能由不同的 λ 项组成，但是只要 Fx=Gx 对所有的 x 都成立，那么 F 和 G 是相等的。

以 F=λx.x 和 G=λx.(λy.y)x 来说，F 是恒等函数，而 G 则是在输入参数 x 上应用恒等函数。F 和 G 虽然由不同的 λ 项组成，但是它们的行为是一样，本质上都是恒等函数。我们称之为 F 和 G 是 η 等价的，F 是 G 的 η 约简，而 G 是 F 的 η 扩展。F 和 G 互为对方的 η 变换。

### λ 演算的推导魔力-邱奇数

邱奇提出了一种函数化数字的方法:邱奇数。

![church-numerals](church-numerals.svg)

表示自然数 $\displaystyle n$ 的高阶函数是個任意函数 $\displaystyle f$ 映射到它自身的n重函数复合之函数，简而言之，数的值即等价于参数被函数包裹的次数。

![church-func](church-func.svg)

以加法函数为例,对plus m n 应用正则序求值：`plus=λmn.m+n = λmn.λFx.m F (n F x)`, 这一步转换很关键，将加法转换为嵌套函数 $F^{m+n}(x)=F^{m}(F^{n}(x))$

### Y 组合子

使用Y组合子实现递归,Y组合子的用处是使得lambda表达式不需要名字。

$$let \space Y = λf.(λx.f(x \space x))(λx.f(x \space x))$$ 

用一个例子函数g来展开它, 

(Y g) = (λf.(λx.(f (x x)) λx.(f (x x))) g)
1. => (λx.g(x x))(λx.g(x x))  // β-归约 - 应用主函数于g
2. => (λy.(g(y y)) λx.(g (x x))) //α-转换 - 重命名约束变量
3. => λy.g(y y) {x: λx.g(x x) } //λy的β-归约 - 应用左侧函数于右侧函数
4. => g((λx.g(x x)) (λx.g(x x))) // 根据第一步的值替换
5. => g( Y g)

Y 组合子推导过程：

以斐波拉契数列为例：

```
let F = λ n. n==0 ? 1 : n*(F n-1)
// F 3
```

可以看到，递归需要我们自己显式调用(非匿名lambda)，有没有更一般的处理方法？有，我们将函数自身作为参数，传给lambda演算，即使用匿名lambda。

```
let F = λ f. λ n. n==0 ? 1 : n*((f f) (n-1))
// (f f) = f,  f需要支持应用于自身。
//F F 3

// 进一步抽象
let gen = λ self. AnyFunction(self self)
// gen gen => AnyFunction(gen gen) 此时能不断生成嵌套的AnyFunction

// gen 规约
let gen = λ self. f(self self)
let Y = λ f.gen(gen)
      = λ f.(λ x.f(x x))(λ x.f(x x))
```

举个例子:

```java
public class YCombinator {

    // T function returning a T
    // T -> T
    // 函数逻辑
    public  interface Func<T>{
        T apply(T n);
    }
    // Higher-order function returning a T function
    // F: F -> (T -> T)
    // 函数的自应用
    private interface Func2TFunc<T> {
        Func<T> apply(Func2TFunc<T> x);
    }

    // We define the Y combinator, apply it to the factorial input function, and apply the result to the input argument. The result is the factorial.
    //  λf.(λx.f(x x))(λx.f(x x))
    // change f->r,x->f get λr.(λf.(f f)) λf.(r λx.((f f) x))  
    public static <T> Func<T> Y(final Func<Func<T>> r) {
        return ((Func2TFunc<T>) f -> f.apply(f))
                .apply(
                        f -> r.apply(
                                x -> f.apply(f).apply(x)));
    }

    public static void main(String args[]) {
        System.out.println(
                // Y combinator
                Y(
                        // Recursive function generator
                        new Func<Func<Integer>>() {
                            @Override
                            public Func<Integer> apply(final Func<Integer> f) {
                                return n -> n == 0 ? 1 : n * f.apply(n - 1);
                            }
                        }

                ).apply(
                        // Argument

                3));
        // 3! =6
    }
}

```

### S K I组合子

我们来看看三个简单的组合子：

* S：S是一个函数应用组合子： S = λ x y z . x z (y z)
* K：K生成一个返回特定常数值的函数： K = λ x . (λ y . x)。 （即扔掉第二个参数，返回第一个参数）
* I：恒等函数： I = λ x . x

I 是可以由S和K表示的：

```
S K K x = 
K x (K x) =   
x =
I x
```

注意，使用S K K，我们创建了I的等价，然而它并没有规约为λ x . x。这实际上符合了η 变换，即两个λ 项外延等价。

> 所有lambda表达式都可以转换为SKI组合子演算,也可以说是 SK组合，此处就不赘述了。

### 类型化

目前为止上面讨论的都是简单的无类型lambda演算，当然也可以说是只有一个类型的lambda演算特例。类型化lambda演算的主要变化是增加了一个叫做 基类型（base types）的概念。例如，我们可以有一个类型 N，它由包含了自然数集合，也可以有一个类型B，对应布尔值true / false，以及一个对应于字符串类型的类S。

函数将一种类型（参数的类型）的值映射到的第二种类型（返回值的类型）的值。对于一个接受类型A的输入参数，并且返回类型B的值的函数，我们将它的类型写为A -> B 。「 ->」叫做函数类型构造器（function type constructor），它是右关联的，所以 A -> B -> C 表示 A -> (B -> C)。

我们添加了一个「:」符号； 冒号左侧是表达式或变量的绑定，其右侧是类型规范。 它表明，其左侧拥有其右侧指定的类型。举几个例子：

* lambda x : N . x + 3。表示参数x 类型为N ，即自然数。这里没有指明函数的结果的类型；但我们知道，函数「+」的类型是 N -> N ，于是可以推断，函数结果的类型是N。
* (lambda x . x + 3) : N -> N，这和上面一样，但类型声明被提了出来，所以它给出了lambda表达式作为一个整体的类型。这一次我们可以推出 x : N ，因为该函数的类型为 N -> N，这意味着该函数参数的类型为 N 。
* lambda x : N, y : B . if y then x * x else x。这是个两个参数的函数，第一个参数类型是 N ，第二个的类型是 B 。我们可以推断返回类型为 N 。于是整个函数的类型是 N -> B -> N 。乍看之下有点奇怪；但请记住，lambda演算实际上只有单个参数；多参数函数的写法只是柯里化的简写。所以实际上这个函数是：lambda x : N . (lambda y : B . if y then x * x else x)；内层lambda的类型是 B -> N ; 外层类型是 N -> (B -> N)。

现在我们得到了一个简单的类型化lambda演算。说它是简单的类型化，是因为这里对类型的处理方式很少：建立新类型的唯一途径就是通过「 ->」 构造器。其他的类型化lambda演算包括了定义「参数化类型」（parametric types）的能力，它将类型表示为不同类型的函数。

## 函数式编程的重要特性

### 高阶函数

正如上面提到的,函数式编程起源于λ 演算。因此，函数式编程的一个最重要的特性之一就是高阶函数。高阶函数以其他函数作为输入，或产生其他函数作为输出。高阶函数使得函数的组合成为可能，更有利于函数的复用。map-reduce就是常见的高阶函数。

### 惰性求值

`g (f input)` 函数f 接受input，f的输出将作为函数g的输入。对于FP而言，函数f和g严格同步执行，仅当函数g试图读取input时，函数f才开始执行，直到f产生了输出，挂起f，运行g。如果g终止而不读取所有f的输出，那么f将被中止。

### 递归

函数式语言中的迭代（循环）通常通过递归来完成。递归和循环在表达能力上是相同的，只不过命令式编程语言偏向于使用循环，而函数式编程语言偏向于使用递归。递归的优势在于天然适合于那些需要用分治法（divide and conquer）解决的问题，把一个大问题划分成小问题，以递归的方式解决。

### Monad

单子（monad）是函数式编程中的一种抽象数据类型，其特别之处在于，它是用来表示计算而不是数据的。在以函数式风格编写的程序中，单子可以用来组织包含有序操作的过程，或者用来定义任意的控制流（比如处理并发、异常、延续）。

单子的构造包括定义两个操作bind和return，还有一个必须满足若干性质的类型构造器M。

* 类型构造器的作用是从底层的类型中创建出一元类型（monadic type）。如果 M 是 Monad 的名称，而 t 是数据类型，则 M t 是对应的一元类型。
* return 操作把一个普通值 t 通过类型构造器封装在一个容器中，所产生的值的类型是 M t。return 操作的名称来源于 Haskell。不过由于 return 在很多编程语言中是保留关键词，用 unit 做名称更为合适。
* bind 操作的类型声明是 (M t)→(t→M u)→(M u)。该操作接受类型为 M t 的值和类型为 t → M u 的函数来对值进行转换。在进行转换时，bind 操作把原始值从容器中抽取出来，再应用给定的函数进行转换。函数的返回值是一个新的容器值 M u。M u 可以作为下一次转换的起点。多个 bind 操作可以级联起来，形成处理流水线。

> [Haskell 中几种 Monad的介绍](http://learnyouahaskell.com/for-a-few-monads-more)


#### wirter monad 

Writer Monad 的主要作用是在函数调用过程中收集辅助信息，比如日志信息或是性能计数器等。其基本的思想是把副作用中对外部环境的修改聚合起来，从而把副作用从函数中分离出来。聚合的方式取决于所产生的副作用。Writer Monad 除了其本身的类型 T 之外，还有另外一个辅助类型 W，用来表示聚合值。对类型 W 的要求是前面提到的两点，也就是存在传递的组合操作和基本单元。Writer Monad 的 return 操作比较简单，返回的是类型 T 的值 t 和类型 W 的基本单元。而 bind 操作则需要分别转换类型 T 和 W 的值。对于 T 的值，按照 Monad 自身的定义来转换；而对于 W 的值，则使用该类型的传递操作来聚合值。聚合的结果作为转换之后的新的 W 的值。

```java
public class WriterMonad<T> {
 
  private final T value;
  private final List<W> sideEffects;
 
  public WriterMonad(T value, List<W> sideEffects) {
    this.value = value;
    this.sideEffects = sideEffects;
  }
 
  public static <T> WriterMonad<T> unit(T value) {
    return new WriterMonad<>(value, List.of());
  }
 
  public static <T1, T2> WriterMonad<T2> bind(WriterMonad<T1> input,
      Function<T1, WriterMonad<T2>> transform) {
    final WriterMonad<T2> result = transform.apply(input.value);
    List<W> sideEffects = new ArrayList<>(input.sideEffects);
    sideEffects.addAll(result.sideEffects);
    return new WriterMonad<>(result.value, sideEffects);
  }
 
  public static <T> WriterMonad<T> pipeline(WriterMonad<T> monad,
      List<Function<T, WriterMonad<T>>> transforms) {
    WriterMonad<T> result = monad;
    for (Function<T, WriterMonad<T>> transform : transforms) {
      result = bind(result, transform);
    }
    return result;
  }
 
  public static void main(String[] args) {
    Function<Integer, WriterMonad<Integer>> transform1 =
        v -> new WriterMonad<>(v * 4, List.of(v + " * 4"));
    Function<Integer, WriterMonad<Integer>> transform2 =
        v -> new WriterMonad<>(v / 2, List.of(v + " / 2"));
    final WriterMonad<Integer> result = 
pipeline(WriterMonad.unit(8),
        List.of(transform1, transform2));
    System.out.println(result); // 输出为 WriterMonad{value=16, 
sideEffects=[8 * 4, 32 / 2]}
  }
}
```

#### reader monad

Reader Monad 也被称为 Environment Monad，描述的是依赖共享环境的计算。Reader Monad 的类型构造器从类型 T 中创建出一元类型 E → T，而 E 是环境的类型。类型构造器把类型 T 转换成一个从类型 E 到 T 的函数。Reader Monad 的 unit 操作把类型 T 的值 t 转换成一个永远返回 t 的函数，而忽略类型为 E 的参数；bind 操作在转换时，在所返回的函数的函数体中对类型 T 的值 t 进行转换，同时保持函数的结构不变。

```java
public class ReaderMonad {
 
  public static <T, E> Function<E, T> unit(T value) {
    return e -> value;
  }
 
  public static <T1, T2, E> Function<E, T2> bind(Function<E, T1>
input, Function<T1, Function<E, T2>> transform) {
    return e -> transform.apply(input.apply(e)).apply(e);
  }
 
  public static void main(String[] args) {
    Function<Environment, String> m1 = unit("Hello");
    Function<Environment, String> m2 = bind(m1, value -> e -> 
e.getPrefix() + value);
    Function<Environment, Integer> m3 = bind(m2, value -> e -> 
e.getBase() + value.length());
    int result = m3.apply(new Environment());
    System.out.println(result);
  }
}
class Environment {
  public String getPrefix() {
    return "##";
  }
 
  public int getBase() {
    return 10000;
  }
}
```

#### state monad

State Monad 可以在计算中附加任意类型的状态值。State Monad 与 Reader Monad 相似，只是 State Monad 在转换时会返回一个新的状态对象，从而可以描述可变的环境。State Monad 的类型构造器从类型 T 中创建一个函数类型，该函数类型的参数是状态对象的类型 S，而返回值包含类型 S 和 T 的值。State Monad 的 unit 操作返回的函数只是简单地返回输入的类型 S 的值；bind 操作所返回的函数类型负责在执行时传递正确的状态对象。

```java
public class StateMonad {
 
  public static <T, S> Function<S, Tuple2<T, S>> unit(T value) {
    return s -> Tuple.of(value, s);
  }
 
  public static <T1, T2, S> Function<S, Tuple2<T2, S>> 
bind(Function<S, Tuple2<T1, S>> input,
      Function<T1, Function<S, Tuple2<T2, S>>> transform) {
    return s -> {
      Tuple2<T1, S> result = input.apply(s);
      return transform.apply(result._1).apply(result._2);
    };
  }
 
  public static void main(String[] args) {
    Function<String, Function<String, Function<State, Tuple2<String, 
State>>>> transform =
        prefix -> value -> s -> Tuple
            .of(prefix + value, new State(s.getValue() + 
value.length()));
 
    Function<State, Tuple2<String, State>> m1 = unit("Hello");
    Function<State, Tuple2<String, State>> m2 = bind(m1, 
transform.apply("1"));
    Function<State, Tuple2<String, State>> m3 = bind(m2, 
transform.apply("2"));
    Tuple2<String, State> result = m3.apply(new State(0));
    System.out.println(result);
  }
}
```

**Monad 在实际中常常是组合使用，也被称为组合子**，在编译器前端中的Parse Combinators,就是一个组合子的使用例子，haskell和scala都有其官方实现。

## 参考

- [1] [Functional Programming's Wiki](https://en.wikipedia.org/wiki/Functional_programming)
- [2] [why functional programming matters](https://www.cs.kent.ac.uk/people/staff/dat/miranda/whyfp90.pdf)
- [3] [how functional programming mattered](https://www.cs.rice.edu/~javaplt/411/19-spring/Readings/WhyFunctionalProrammingMattered.pdf)
- [4] [Lamda Calculus](http://goodmath.blogspot.com/2006/06/lamda-calculus-index.html)
- [5] [函数式编程思想概论](https://www.ibm.com/developerworks/cn/java/j-understanding-functional-programming-1/index.html)
- [6] [邱奇数](https://zh.wikipedia.org/wiki/%E9%82%B1%E5%A5%87%E6%95%B0)
- [7] [The Y Combinator](https://eecs.ceas.uc.edu/~franco/C511/html/Scheme/ycomb.html)
- [8] [Java Y Combinator](http://www.righto.com/2009/03/y-combinator-in-arc-and-java.html)

