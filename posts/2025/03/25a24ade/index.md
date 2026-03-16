# Rust 智能指针简介

指针是一个包含了内存地址的变量，该内存地址引用或者指向了另外的数据。

在 Rust 中，最常见的指针类型是引用，引用通过 & 符号表示。不同于其它语言，引用在 Rust 中被赋予了更深层次的含义，那就是：借用其它变量的值。引用本身很简单，除了指向某个值外并没有其它的功能，也不会造成性能上的额外损耗，因此是 Rust 中使用最多的指针类型。

而智能指针则不然，它虽然也号称指针，但是它是一个复杂的家伙：通过比引用更复杂的数据结构，包含比引用更多的信息，例如元数据，当前长度，最大可用长度等。智能指针往往是基于结构体实现，它与我们自定义的结构体最大的区别在于它实现了 Deref 和 Drop 特征：

- Deref 可以让智能指针像引用那样工作，这样你就可以写出同时支持智能指针和引用的代码，例如 *T
- Drop 允许你指定智能指针超出作用域后自动执行的代码，例如做一些数据清除等收尾工作

而 Box 指针是最简单的智能指针。本文将介绍 Box 指针以及 Deref 和 Drop 特征。

<!--more-->

## Box 指针

栈内存从高位地址向下增长，且栈内存是连续分配的，一般来说操作系统对栈内存的大小都有限制，因此 C 语言中无法创建任意长度的数组。在 Rust 中，main 线程的栈大小是 8MB，普通线程是 2MB，在函数调用时会在其中创建一个临时栈空间，调用结束后 Rust 会让这个栈空间里的对象自动进入 Drop 流程，最后栈顶指针自动移动到上一个调用栈顶，无需程序员手动干预，因而栈内存申请和释放是非常高效的。

与栈相反，堆上内存则是从低位地址向上增长，堆内存通常只受物理内存限制，而且通常是不连续的，因此从性能的角度看，栈往往比堆更高。但是并不绝对。

- 小型数据，在栈上的分配性能和读取性能都要比堆上高
- 中型数据，栈上分配性能高，但是读取性能和堆上并无区别，因为无法利用寄存器或 CPU 高速缓存，最终还是要经过一次内存寻址
- 大型数据，只建议在堆上分配和使用

总之，栈的分配速度肯定比堆上快，但是读取速度往往取决于你的数据能不能放入寄存器或 CPU 高速缓存。 因此不要仅仅因为堆上性能不如栈这个印象，就总是优先选择栈，导致代码更复杂的实现。

由于 Box智能指针(实现了 Deref 和 Drop 特征)是简单的封装，除了将值存储在堆上外，并没有其它性能上的损耗。而性能和功能往往是鱼和熊掌，因此 Box 相比其它智能指针，功能较为单一，可以在以下场景中使用它：

- 特意的将数据分配在堆上
- 数据较大时，又不想在转移所有权时进行数据拷贝
- 类型的大小在编译期无法确定，但是我们又需要固定大小的类型时
- 特征对象，用于说明对象实现了一个特征，而不是某个特定的类型

### 使用 `Box<T>` 将数据存储在堆上

如果一个变量拥有一个数值 let a = 3，那变量 a 必然是存储在栈上的，那如果我们想要 a 的值存储在堆上就需要使用`Box<T>`：

```rust
fn main() {
    let a = Box::new(3);
    println!("a = {}", a); // a = 3

    let b = *a + 1; // 改为使用下面代码将报错
    // let b = a + 1; // cannot add `{integer}` to `Box<{integer}>`
}
```

let b = a + 1 报错，是因为在表达式中，我们无法自动隐式地执行 Deref 解引用操作，你需要使用 * 操作符 let b = *a + 1，来显式的进行解引用。

### 避免栈上数据的拷贝

当栈上数据转移所有权时，实际上是把数据拷贝了一份，最终新旧变量各自拥有不同的数据，因此所有权并未转移。

而堆上则不然，底层数据并不会被拷贝，转移所有权仅仅是复制一份栈中的指针，再将新的指针赋予新的变量，然后让拥有旧指针的变量失效，最终完成了所有权的转移：

```rust
fn main() {
    // 在栈上创建一个长度为1000的数组
    let arr = [0;1000];
    // 将arr所有权转移arr1，由于 `arr` 分配在栈上，因此这里实际上是直接重新深拷贝了一份数据
    let arr1 = arr;

    // arr 和 arr1 都拥有各自的栈上数组，因此不会报错
    println!("{:?}", arr.len());
    println!("{:?}", arr1.len());

    // 在堆上创建一个长度为1000的数组，然后使用一个智能指针指向它
    let arr = Box::new([0;1000]);
    // 将堆上数组的所有权转移给 arr1，由于数据在堆上，因此仅仅拷贝了智能指针的结构体，底层数据并没有被拷贝
    // 所有权顺利转移给 arr1，arr 不再拥有所有权
    let arr1 = arr;
    println!("{:?}", arr1.len());
    // 由于 arr 不再拥有底层数组的所有权，因此下面代码将报错
    // println!("{:?}", arr.len());
}
```

### 将动态大小类型变为 Sized 固定大小类型

Rust 需要在编译时知道类型占用多少空间，如果一种类型在编译时无法知道具体的大小，那么被称为动态大小类型 DST。

其中一种无法在编译时知道大小的类型是递归类型：在类型定义中又使用到了自身，或者说该类型的值的一部分可以是相同类型的其它值，这种值的嵌套理论上可以无限进行下去，所以 Rust 不知道递归类型需要多少空间：

```rust
enum List {
    Cons(i32, List),
    Nil,
}
```

以上就是函数式语言中常见的 Cons List，它的每个节点包含一个 i32 值，还包含了一个新的 List，因此这种嵌套可以无限进行下去，Rust 认为该类型是一个 DST 类型，并给予报错：

```rust
error[E0072]: recursive type `List` has infinite size //递归类型 `List` 拥有无限长的大小
 --> src/main.rs:3:1
  |
3 | enum List {
  | ^^^^^^^^^ recursive type has infinite size
4 |     Cons(i32, List),
  |               ---- recursive without indirection
```

此时若想解决这个问题，就可以使用我们的 Box<T>：

```rust
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```
只需要将 List 存储到堆上，然后使用一个智能指针指向它，即可完成从 DST 到 Sized 类型(固定大小类型)的华丽转变。

### 特征对象

在 Rust 中，想实现不同类型组成的数组只有两个办法：枚举和特征对象，前者限制较多，因此后者往往是最常用的解决办法。

```rust
trait Draw {
    fn draw(&self);
}

struct Button {
    id: u32,
}
impl Draw for Button {
    fn draw(&self) {
        println!("这是屏幕上第{}号按钮", self.id)
    }
}

struct Select {
    id: u32,
}

impl Draw for Select {
    fn draw(&self) {
        println!("这个选择框贼难用{}", self.id)
    }
}

fn main() {
    let elems: Vec<Box<dyn Draw>> = vec![Box::new(Button { id: 1 }), Box::new(Select { id: 2 })];

    for e in elems {
        e.draw()
    }
}
```
> 以上代码将不同类型的 Button 和 Select 包装成 Draw 特征的特征对象，放入一个数组中，`Box<dyn Draw>`就是特征对象。其实，特征也是 DST 类型，而特征对象在做的就是将 DST 类型转换为固定大小类型。

### Box 内存布局

先来看看 Vec<i32> 的内存布局：

```
(stack)    (heap)
┌──────┐   ┌───┐
│ vec1 │──→│ 1 │
└──────┘   ├───┤
           │ 2 │
           ├───┤
           │ 3 │
           ├───┤
           │ 4 │
           └───┘
```

Vec 和 String 都是智能指针，从上图可以看出，该智能指针存储在栈中，然后指向堆上的数组数据。那如果数组中每个元素都是一个 Box 对象呢？来看看 `Vec<Box<i32>>` 的内存布局：

```
                    (heap)
(stack)    (heap)   ┌───┐
┌──────┐   ┌───┐ ┌─→│ 1 │
│ vec2 │──→│B1 │─┘  └───┘
└──────┘   ├───┤    ┌───┐
           │B2 │───→│ 2 │
           ├───┤    └───┘
           │B3 │─┐  ┌───┐
           ├───┤ └─→│ 3 │
           │B4 │─┐  └───┘
           └───┘ │  ┌───┐
                 └─→│ 4 │
                    └───┘
```

上面的 B1 代表被 Box 分配到堆上的值 1。

可以看出智能指针 vec2 依然是存储在栈上，然后指针指向一个堆上的数组，该数组中每个元素都是一个 Box 智能指针，最终 Box 智能指针又指向了存储在堆上的实际值。

因此当我们从数组中取出某个元素时，取到的是对应的智能指针 Box，需要对该智能指针进行解引用，才能取出最终的值：
```rust
fn main() {
    let arr = vec![Box::new(1), Box::new(2)];
    let (first, second) = (&arr[0], &arr[1]);
    let sum = **first + **second;
}
```
以上代码有几个值得注意的点：

- 使用 `&` 借用数组中的元素，否则会报所有权错误.
- 表达式不能隐式的解引用，因此必须使用 `**` 做两次解引用，第一次将 `&Box<i32>` 类型转成 `Box<i32>`，第二次将 `Box<i32>` 转成 i32.

### Box::leak
Box 中还提供了一个非常有用的关联函数：`Box::leak`，它可以消费掉 Box 并且强制目标值从内存中泄漏，例如，你可以把一个 String 类型，变成一个 `'static` 生命周期的 `&str` 类型：

```rust
fn main() {
   let s = gen_static_str();
   println!("{}", s);
}

fn gen_static_str() -> &'static str{
    let mut s = String::new();
    s.push_str("hello, world");

    Box::leak(s.into_boxed_str())
}
```
在之前的代码中，如果 String 创建于函数中，那么返回它的唯一方法就是转移所有权给调用者 `fn move_str() -> String`，而通过 `Box::leak` 我们不仅返回了一个 `&str` 字符串切片，它还是 `'static`生命周期的！真正具有 'static 生命周期的往往都是编译期就创建的值，例如 let v = "hello, world"，这里 v 是直接打包到二进制可执行文件中的，因此该字符串具有 `'static`生命周期，再比如 const 常量。

## Deref 解引用

Deref 可以让智能指针像引用那样工作，这样你就可以写出同时支持智能指针和引用的代码，例如 `*T`。考虑一下智能指针，它是一个结构体类型，如果你直接对它进行`*myStruct`，显然编译器不知道该如何办，因此我们可以为智能指针结构体实现 Deref 特征。实现 Deref 后的智能指针结构体，就可以像普通引用一样，通过 * 进行解引用，例如 Box<T> 智能指针：

```rust
fn main() {
    let x = Box::new(1);
    let sum = *x + 1;
}
```

智能指针 x 被 `*` 解引用为 i32 类型的值 1，然后再进行求和。

### 定义自己的智能指针

现在，让我们一起来实现一个智能指针。并实现 Deref 特征，以支持 `*` 解引用操作符(标准库实现的智能指针要考虑很多边边角角情况，肯定比我们的实现要复杂)。

。

```rust
use std::ops::Deref;

struct MyBox<T>(T);

impl<T> MyBox<T> {
    fn new(x: T) -> MyBox<T> {
        MyBox(x)
    }
}

impl<T> Deref for MyBox<T> {
    type Target = T;
    fn deref(&self) -> &Self::Target {
        &self.0
    }
}
```

当解引用 MyBox 智能指针时，返回元组结构体中的元素 `&self.0`，有几点要注意的：

- 在 Deref 特征中声明了关联类型 Target，在之前章节中介绍过，关联类型主要是为了提升代码可读性。
- deref 返回的是一个常规引用，可以被 * 进行解引用。

当我们对智能指针 Box 进行解引用时，实际上 Rust 为我们调用了以下方法：

```rust
*(y.deref())
```

首先调用 deref 方法返回值的常规引用，然后通过 `*` 对常规引用进行解引用，最终获取到目标值。这种实现的原因在于所有权系统的存在。如果 deref 方法直接返回一个值，而不是引用，那么该值的所有权将被转移给调用者，而我们不希望调用者仅仅只是`*T`一下，就拿走了智能指针中包含的值。

需要注意的是，`*` 不会无限递归替换，从 `*y` 到 `*(y.deref())` 只会发生一次，而不会继续进行替换然后产生形如 `*((y.deref()).deref())` 的怪物。

### 隐式 Deref 转换

对于函数和方法的传参，Rust 提供了一个极其有用的隐式转换：Deref 转换。若一个类型实现了 Deref 特征，那它的引用在传给函数或方法时，会根据参数签名来决定是否进行隐式的 Deref 转换，例如：

```rust
fn main() {
    let s = String::from("hello world");
    display(&s)
}
fn display(s: &str) {
    println!("{}",s);
}
/// string
#[stable(feature = "rust1", since = "1.0.0")]
impl ops::Deref for String {
    type Target = str;
    #[inline]
    fn deref(&self) -> &str {
        self.as_str()
    }
}
```
以上代码有几点值得注意：

- String 实现了 Deref 特征，可以在需要时自动被转换为 &str 类型
- &s 是一个 &String 类型，当它被传给 display 函数时，自动通过 Deref 转换成了 &str
- 必须使用 &s 的方式来触发 Deref(**仅引用类型的实参才会触发自动解引用**)

Deref 可以支持连续的隐式转换，直到找到适合的形式为止[^1]：

```rust
fn main() {
    let s = MyBox::new(String::from("hello world"));
    display(&s)
}
fn display(s: &str) {
    println!("{}",s);
}
```

再来看一下在方法、赋值中自动应用 Deref 的例子[^2]：

```rust
fn main() {
    let s = MyBox::new(String::from("hello, world"));
    let s1: &str = &s;
    let s2: String = s.to_string();
}
```

### 总结

一个类型为 `T` 的对象 foo，如果 `T: Deref<Target=U>`，那么，相关 foo 的引用 `&foo` 在应用的时候会自动转换为 `&U`。

Rust 编译器实际上只能对 `&v` 形式的引用进行解引用操作，那么如果是一个智能指针或者 `&&&&v` 类型, 该如何对这两个进行解引用？

Rust 会在解引用时自动把智能指针和 `&&&&v` 做引用归一化操作，转换成 `&v` 形式，最终再对 `&v` 进行解引用：

- 把智能指针（比如在库中定义的，Box、Rc、Arc、Cow 等）从结构体脱壳为内部的引用类型，也就是转成结构体内部的 &v
- 把多重&，例如 `&&&&&&&v`，归一成 `&v`

关于第二种情况，我们来看一段标准库源码：

```rust
impl<T: ?Sized> Deref for &T {
    type Target = T;

    fn deref(&self) -> &T {
        *self
    }
}
```

在这段源码中，`&T` 被自动解引用为 `T`，也就是 `&T: Deref<Target=T>` 。 按照这个代码，`&&&&T` 会被自动解引用为 `&&&T`，然后再自动解引用为 `&&T`，以此类推， 直到最终变成 `&T`。

Rust 还支持将一个可变的引用转换成另一个可变的引用以及将一个可变引用转换成不可变的引用，规则如下：

- 当 `T: Deref<Target=U>`，可以将 `&T` 转换成 `&U`
- 当 `T: DerefMut<Target=U>`，可以将 `&mut T` 转换成 `&mut U`
- 当 `T: Deref<Target=U>`，可以将 `&mut T` 转换成 `&U`

```rust
use std::ops::Deref;
use std::ops::DerefMut;

impl<T> Deref for MyBox<T> {
    type Target = T;

    fn deref(&self) -> &Self::Target {
        &self.v
    }
}
impl<T> DerefMut for MyBox<T> {
    fn deref_mut(&mut self) -> &mut Self::Target {
        &mut self.v
    }
}
fn main() {
    let mut s = MyBox::new(String::from("hello, "));
    display(&mut s)
}
fn display(s: &mut String) {
    s.push_str("world");
    println!("{}", s);
}
```

- 要实现 DerefMut 必须要先实现 Deref 特征：`pub trait DerefMut: Deref`
- 可变的引用转换成另一个可变的引用，对应上例中，就是将 `&mut MyBox<String>` 转换为 `&mut String`
- 上述三条规则中的第三条，它比另外两条稍微复杂了点：Rust 可以把可变引用隐式的转换成不可变引用，但反之则不行[^3]。

> 我们也可以为自己的类型实现 Deref 特征，但是原则上来说，只应该为自定义的智能指针实现 Deref

## Drop 释放资源

Drop 特征: 在 Rust 中，你可以指定在一个变量超出作用域时，执行一段特定的代码，最终编译器将帮你自动插入这段收尾代码 ^[Drop特征中的drop方法借用了目标的可变引用，而不是拿走了所有权]。先来看一个例子:

```rust
struct HasDrop1;
struct HasDrop2;
impl Drop for HasDrop1 {
    fn drop(&mut self) {
        println!("Dropping HasDrop1!");
    }
}
impl Drop for HasDrop2 {
    fn drop(&mut self) {
        println!("Dropping HasDrop2!");
    }
}
struct HasTwoDrops {
    one: HasDrop1,
    two: HasDrop2,
}
impl Drop for HasTwoDrops {
    fn drop(&mut self) {
        println!("Dropping HasTwoDrops!");
    }
}
struct Foo;
impl Drop for Foo {
    fn drop(&mut self) {
        println!("Dropping Foo!")
    }
}
fn main() {
    let _x = HasTwoDrops {   -----| _x 所有权
        two: HasDrop2,            |
        one: HasDrop1,            |
    };                            |
    let _foo = Foo;          -----| _foo 所有权
    println!("Running!");         |
}                            -----| 所有权结束
/// Running!
/// Dropping Foo!
/// Dropping HasTwoDrops!
/// Dropping HasDrop1!
/// Dropping HasDrop2!
```
观察以上输出，我们可以得出以下关于 Drop 顺序的结论 ^[实际上,就算你不为`_x`结构体实现Drop特征，它内部的两个字段依然会调用drop,Rust自动为几乎所有类型都实现了Drop特征]

- 变量级别，按照逆序的方式，_x 在 _foo 之前创建，因此 _x 在 _foo 之后被 drop
- 结构体内部，按照顺序的方式，结构体 _x 中的字段按照定义中的顺序依次 drop

### 手动回收

当使用智能指针来管理锁的时候，你可能希望提前释放这个锁，然后让其它代码能及时获得锁，此时就需要提前去手动 drop。 但是在之前我们提到一个悬念，Drop::drop 只是借用了目标值的可变引用，所以，就算你提前调用了 drop，后面的代码依然可以使用目标值，但是这就会访问一个并不存在的值，非常不安全，好在 Rust 会阻止你：

```rust
#[derive(Debug)]
struct Foo;
impl Drop for Foo {
    fn drop(&mut self) {
        println!("Dropping Foo!")
    }
}
fn main() {
    let foo = Foo;
    foo.drop();
    println!("Running!:{:?}", foo);
}
```

报错信息如下：

```
error[E0040]: explicit use of destructor method
  --> src/main.rs:37:9
   |
37 |     foo.drop();
   |     ----^^^^--
   |     |   |
   |     |   explicit destructor calls not allowed
   |     help: consider using `drop` function: `drop(foo)`
```

编译器直接阻止了我们调用 Drop 特征的 drop 方法，原因是对于 Rust 而言，不允许显式地调用析构函数（这是一个用来清理实例的通用编程概念）。但是可以使用`std::mem::drop` 函数。

```rust
fn main() {
    let foo = Foo;
    drop(foo);
    // 以下代码会报错：借用了所有权被转移的值
    // println!("Running!:{:?}", foo);
}
```

>事实上，能被显式调用的drop(_x)函数只是个空函数，在拿走目标值的所有权后没有任何操作。而由于其持有目标值的所有权，在drop(_x)函数结束之际，编译器会执行_x真正的析构函数，从而完成释放资源的操作。换句话说，drop(_x)函数只是帮助目标值的所有者提前离开了作用域。

### 使用场景

对于 Drop 而言，主要有两个功能：

- 回收内存资源: 在绝大多数情况下，我们都无需手动去 drop 以回收内存资源，因为 Rust 会自动帮我们完成这些工作，它甚至会对复杂类型的每个字段都单独的调用 drop 进行回收！但是确实有极少数情况，需要你自己来回收资源的，例如文件描述符、网络 socket 等，当这些值超出作用域不再使用时，就需要进行关闭以释放相关的资源，在这些情况下，就需要使用者自己来解决 Drop 的问题。
- 执行一些收尾工作: 指定在一个变量超出作用域时，执行一段特定的代码，最终编译器将帮你自动插入这段收尾代码。

> 我们无法为一个类型同时实现 Copy 和 Drop 特征。因为实现了 Copy 特征的类型会被编译器隐式的复制，因此非常难以预测析构函数执行的时间和频率。因此这些实现了 Copy 的类型无法拥有析构函数。

## 参考

- [1] [Rust语言圣经#4.4智能指针](https://course.rs/advance/smart-pointer/intro.html)

--- 

[^1]: 这里我们使用了之前自定义的智能指针 MyBox，并将其通过连续的隐式转换变成 &str 类型：首先 MyBox 被 Deref 成 String 类型，结果并不能满足 display 函数参数的要求，编译器发现 String 还可以继续 Deref 成 &str，最终成功的匹配了函数参数。

[^2]: 对于 s1，我们通过两次 Deref 将 &str 类型的值赋给了它（赋值操作需要手动解引用）；而对于 s2，我们在其上直接调用方法 to_string，实际上 MyBox 根本没有没有实现该方法，能调用 to_string，完全是因为编译器对 MyBox 应用了 Deref 的结果(**方法调用会自动解引用**)。

[^3]: 如果从 Rust 的所有权和借用规则的角度考虑，当你拥有一个可变的引用，那该引用肯定是对应数据的唯一借用，那么此时将可变引用变成不可变引用并不会破坏借用规则；但是如果你拥有一个不可变引用，那同时可能还存在其它几个不可变的引用，如果此时将其中一个不可变引用转换成可变引用，就变成了可变引用与不可变引用的共存，最终破坏了借用规则。

