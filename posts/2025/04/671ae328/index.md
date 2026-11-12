# Rust-生命周期

生命周期，简而言之就是引用的有效作用域。在大多数时候，我们无需手动的声明生命周期，因为编译器可以自动进行推导。当多个生命周期存在，且编译器无法推导出某个引用的生命周期时，就需要我们手动标明生命周期。

## 悬垂指针和生命周期

生命周期的主要作用是避免悬垂引用，它会导致程序引用了本不该引用的数据：

```rust
{
    let r;
    {
        let x = 5;
        r = &x;
    }    // ^^ borrowed value does not live long enough
    println!("r: {}", r);
}
```

此处 r 就是一个悬垂指针，它引用了提前被释放的变量 x。

## 借用检查

为了保证 Rust 的所有权和借用的正确性，Rust 使用了一个借用检查器(Borrow checker)，来检查我们程序的借用正确性：

```rust
{
    let r;                // ---------+-- 'a
                          //          |
    {                     //          |
        let x = 5;        // -+-- 'b  |
        r = &x;           //  |       |
    }                     // -+       |
                          //          |
    println!("r: {}", r); //          |
}                         // ---------+
```

r 变量被赋予了生命周期 'a，x 被赋予了生命周期 'b，从图示上可以明显看出生命周期 'b 比 'a 小很多。在编译期，Rust 会比较两个变量的生命周期，结果发现 r 明明拥有生命周期 'a，但是却引用了一个小得多的生命周期 'b，在这种情况下，编译器会认为我们的程序存在风险，因此拒绝运行。如果想要编译通过，也很简单，只要 'b 比 'a 大就好。总之，x 变量只要比 r 活得久，那么 r 就能随意引用 x 且不会存在危险：

```rust
{
    let x = 5;            // ----------+-- 'b
                          //           |
    let r = &x;           // --+-- 'a  |
                          //   |       |
    println!("r: {}", r); //   |       |
                          // --+       |
}                         // ----------+
```
<!--more-->

## 生命周期标注

编译器有时会无法自动推导生命周期，此时就需要我们手动去标注，来帮助编译器进行借用检查的分析。生命周期标注并不会改变任何引用的实际作用域，只是帮助编译器分析生命周期。生命周期的语法以 ' 开头，名称往往是一个单独的小写字母，例如`'a`。 如果是引用类型的参数，那么生命周期会位于引用符号 & 之后，并用一个空格来将生命周期和引用参数分隔开。

```rust
&i32        // 一个引用
&'a i32     // 具有显式生命周期的引用
&'a mut i32 // 具有显式生命周期的可变引用
```

### 生命周期约束语法

对于两个生命周期，可以通过生命周期约束语法来描述大小。`'a: 'b`用于说明 'a 必须比 'b 活得久。

```rust
struct DoubleRef<'a,'b:'a, T> {
    r: &'a T,
    s: &'b T
}
```


同样`T: 'a` 表示类型 T 必须比 'a 活得要久(因为结构体字段 r 引用了 T，因此 r 的生命周期 'a 必须要比 T 的生命周期更短，被引用者的生命周期必须要比引用长)：

```rust
struct Ref<'a, T: 'a> {
    r: &'a T
}
```

从 1.31 版本开始，编译器可以自动推导 T: 'a 类型的约束，因此我们只需这样写即可：

```rust
struct Ref<'a, T> {
    r: &'a T
}
```

## 函数中的生命周期

先来考虑一个例子 - 返回两个字符串切片中较长的那个，该函数的参数是两个字符串切片，返回值也是字符串切片。

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

上面的方法在编译时会报错:

```rust
error[E0106]: missing lifetime specifier
 --> src/lib.rs:3:33
  |
3 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
3 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++

For more information about this error, try `rustc --explain E0106`.
error: could not compile `playground` (lib) due to 1 previous error
```

根据提示可以知道，是编译器无法知道该函数的返回值到底引用 x 还是 y ，因为编译器需要知道这些，来确保函数调用后的引用生命周期分析。

推荐修改方式是 `fn longest<'a>(x: &'a str, y: &'a str) -> &'a str`。通过`'a`标注，告诉编译器，参数 `x`，`y` 和返回值至少活得和`'a` 一样久。

换句话说就是，`'a` 的大小将等于 x 和 y 中生命周期较小的那个。由于返回值的生命周期也被标记为 `'a`，因此返回值的生命周期也是 x 和 y 中生命周期较小的那个。

如果一个函数永远只返回第一个参数 x，那么生命周期标注如下：

```rust
fn first<'a>(x: &'a str, y: &str) -> &'a str {
    x
}
```

函数的返回值如果是一个引用类型，那么它的生命周期只会来源于：

- 函数参数的生命周期
- 函数体中某个新建引用的生命周期

若是第二种情况，就是典型的悬垂引用问题：

```rust
fn longest<'a>(x: &str, y: &str) -> &'a str {
    let result = String::from("really long string");
    result.as_str()
}
```
该函数会报错：

```rust
error[E0515]: cannot return value referencing local variable `result`
  --> src/main.rs:11:5
   |
11 |     result.as_str()
   |     ------^^^^^^^^^
   |     |
   |     returns a value referencing data owned by the current function
   |     `result` is borrowed here
```

## 结构体的生命周期

```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}
```

ImportantExcerpt 结构体中有一个引用类型的字段 part，因此需要为它标注上生命周期。结构体的生命周期标注语法跟泛型参数语法很像，需要对生命周期参数进行声明 `<'a>`。该生命周期标注说明，结构体 ImportantExcerpt 所引用的字符串 str 生命周期需要大于等于该结构体的生命周期。

## 生命周期消除

编译器使用三条消除规则来确定哪些场景不需要显式地去标注生命周期。其中第一条规则应用在输入生命周期上，第二、三条应用在输出生命周期上。若编译器发现三条规则都不适用时，就会报错，提示你需要手动标注生命周期。

> 函数或者方法中，参数的生命周期被称为 输入生命周期，返回值的生命周期被称为 输出生命周期。

- 每一个引用参数都会获得独自的生命周期; 例如一个引用参数的函数就有一个生命周期标注: `fn foo<'a>(x: &'a i32)`，两个引用参数的有两个生命周期标注:`fn foo<'a, 'b>(x: &'a i32, y: &'b i32)`, 依此类推。

- 若只有一个输入生命周期(函数参数中只有一个引用类型)，那么该生命周期会被赋给所有的输出生命周期，也就是所有返回值的生命周期都等于该输入生命周期;例如函数`fn foo(x: &i32) -> &i32`，x 参数的生命周期会被自动赋给返回值 &i32，因此该函数等同于 `fn foo<'a>(x: &'a i32) -> &'a i32`
- 若存在多个输入生命周期，且其中一个是 `&self` 或 `&mut self`，则 `&self` 的生命周期被赋给所有的输出生命周期。拥有`&self` 形式的参数，说明该函数是一个方法，该规则让方法的使用便利度大幅提升。若一个方法，它的返回值的生命周期跟参数`&self`的不一样，那么需要手动标注。

### 扩展的消除规则

#### impl 块消除

```rust
impl<'a> Reader for BufReader<'a> {
    // methods go here
    // impl内部实际上没有用到'a
}
// 在 impl 内部的方法中，根本就没有用到 'a，那就可以写成下面的代码形式。
impl Reader for BufReader<'_> {
    // methods go here
}
```

`'_` 称为匿名生命周期（anonymous lifetime），在这里表示 BufReader 有一个不使用的生命周期，我们可以忽略它，无需为它创建一个名称。


### 闭包函数的消除规则

```rust
fn fn_elision(x: &i32) -> &i32 { x }
let closure_slision = |x: &i32| -> &i32 { x };
```

闭包函数编译报错:

```
error: lifetime may not live long enough
  --> src/main.rs:39:39
   |
39 |     let closure = |x: &i32| -> &i32 { x }; // fails
   |                       -        -      ^ returning this value requires that `'1` must outlive `'2`
   |                       |        |
   |                       |        let's call the lifetime of this reference `'2`
   |                       let's call the lifetime of this reference `'1`
```

对于函数的生命周期而言，它的消除规则之所以能生效是因为它的生命周期完全体现在签名的引用类型上，在函数体中无需任何体现：

```rust
fn fn_elision(x: &i32) -> &i32 {..}
```
因此编译器可以做各种编译优化，也很容易根据参数和返回值进行生命周期的分析，最终得出消除规则。可是闭包，并没有函数那么简单，它的生命周期分散在参数和闭包函数体中(主要是它没有确切的返回值签名)：

```rust
let closure_slision = |x: &i32| -> &i32 { x };
```
编译器就必须深入到闭包函数体中，去分析和推测生命周期，复杂度因此急剧提升。使用闭包函数，需要一些额外的工作:

```rust
fn main() {
   let closure_slision = fun(|x: &i32| -> &i32 { x });
   assert_eq!(*closure_slision(&45), 45);
   // Passed !
}
// 提取出闭包的签名
fn fun<T, F: Fn(&T) -> &T>(f: F) -> F {
   f
}
```

## 方法中的生命周期

为具有生命周期的结构体实现方法时，我们使用的语法跟泛型参数语法很相似：


```rust
struct ImportantExcerpt<'a> {
    part: &'a str,
}

impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part(&self, announcement: &str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

首先，编译器应用第一规则，给予每个输入参数一个生命周期:

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```
需要注意的是，编译器不知道 announcement 的生命周期到底多长，因此它无法简单的给予它生命周期 `'a`，而是重新声明了一个全新的生命周期 `'b`。

接着，编译器应用第三规则，将 &self 的生命周期赋给返回值 &str：

```rust
impl<'a> ImportantExcerpt<'a> {
    fn announce_and_return_part<'b>(&'a self, announcement: &'b str) -> &'a str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

如果我们想修改返回的生命周期。

```rust
impl<'a: 'b, 'b> ImportantExcerpt<'a> {
    fn announce_and_return_part(&'a self, announcement: &'b str) -> &'b str {
        println!("Attention please: {}", announcement);
        self.part
    }
}
```

## 静态生命周期

Rust 中有一个非常特殊的生命周期，那就是 'static，拥有该生命周期的引用可以和整个程序活得一样久。字符串常量全部具有 'static 的生命周期：

```rust
let s: &'static str = "static value";
```

> 当生命周期不知道怎么标时，可以施加一个静态生命周期的约束`'static`,可以帮助我们解决非常复杂的生命周期问题甚至是无法被手动解决的生命周期问题。

### `&'static`

`&'static`对于生命周期有着非常强的要求：一个引用必须要活得跟剩下的程序一样久，才能被标注为`&'static`。对于字符串字面量来说，它直接被打包到二进制文件中，永远不会被 drop，因此它能跟程序活得一样久，自然它的生命周期是 `'static`。但是，`&'static` 生命周期针对的仅仅是引用，而不是持有该引用的变量，对于变量来说，还是要遵循相应的作用域规则。

```rust
use std::{slice::from_raw_parts, str::from_utf8_unchecked};

fn get_memory_location() -> (usize, usize) {
  // “Hello World” 是字符串字面量，因此它的生命周期是 `'static`.
  // 但持有它的变量 `string` 的生命周期就不一样了，它完全取决于变量作用域，对于该例子来说，也就是当前的函数范围
  let string = "Hello World!";
  let pointer = string.as_ptr() as usize;
  let length = string.len();
  (pointer, length)
  // `string` 在这里被 drop 释放
  // 虽然变量被释放，无法再被访问，但是数据依然还会继续存活
}

fn get_str_at_location(pointer: usize, length: usize) -> &'static str {
  // 使用裸指针需要 `unsafe{}` 语句块
  unsafe { from_utf8_unchecked(from_raw_parts(pointer as *const u8, length)) }
}

fn main() {
  let (pointer, length) = get_memory_location();
  let message = get_str_at_location(pointer, length);
  println!(
    "The {} bytes at 0x{:X} stored: {}",
    length, pointer, message
  );
}
```


## 无界生命周期

不安全代码(unsafe)经常会凭空产生引用或生命周期，这些生命周期被称为是 无界(unbound) 的。

无界生命周期往往是在解引用一个裸指针(裸指针 raw pointer)时产生的，换句话说，它是凭空产生的，因为输入参数根本就没有这个生命周期：

```rust
fn f<'a, T>(x: *const T) -> &'a T {
    unsafe {
        &*x
    }
}
```

上述代码中，参数 x 是一个裸指针，它并没有任何生命周期，然后通过 unsafe 操作后，它被进行了解引用，变成了一个 Rust 的标准引用类型，该类型必须要有生命周期，也就是 `'a`。可以看出 `'a` 是凭空产生的，因此它是无界生命周期。这种生命周期由于没有受到任何约束，因此它想要多大就多大，这实际上比 `'static` 要强大。

在实际应用中，要尽量避免这种无界生命周期。

## NLL (Non-Lexical Lifetime)

所有权规则是:当所有者（变量）离开作用域范围时，这个值将被丢弃(drop)。因此生命周期正常来说应该从借用开始一直持续到作用域结束，但是这种规则会让多引用共存的情况变得更复杂:

```rust
fn main() {
   let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{} and {}", r1, r2);
    // rust 1.31以后，r1,r2作用域在这里结束

    let r3 = &mut s;
    println!("{}", r3);
}
```

按照上述规则，这段代码将会报错，因为 r1 和 r2 的不可变引用将持续到 main 函数结束，而在此范围内，我们又借用了 r3 的可变引用，这违反了借用的规则：要么多个不可变借用，要么一个可变借用。从 1.31 版本引入 NLL 后，生命周期就变成了：引用的生命周期从借用处开始，一直持续到最后一次使用的地方。因此，上面的代码也就可以通过。

```rust
let mut u = 0;
let mut v = 1;
let mut w = 2;

// lifetime of `a` = α ∪ β ∪ γ
let mut a = &mut u;     // --+ α. lifetime of `&mut u`  --+ lexical "lifetime" of `&mut u`,`&mut u`, `&mut w` and `a`
use(a);                 //   |                            |
*a = 3; // <-----------------+                            |
...                     //                                |
a = &mut v;             // --+ β. lifetime of `&mut v`    |
use(a);                 //   |                            |
*a = 4; // <-----------------+                            |
...                     //                                |
a = &mut w;             // --+ γ. lifetime of `&mut w`    |
use(a);                 //   |                            |
*a = 5; // <-----------------+ <--------------------------+
```

a 有三段生命周期：α，β，γ，每一段生命周期都随着当前值的最后一次使用而结束。

## Reborrow 再借用

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}
impl Point {
    fn move_to(&mut self, x: i32, y: i32) {
        self.x = x;
        self.y = y;
    }
}
fn main() {
    let mut p = Point { x: 0, y: 0 };
    let r = &mut p;
    // reborrow! 此时对`r`的再借用不会导致跟上面的借用冲突
    let rr: &Point = &*r;
    // 再借用`rr`最后一次使用发生在这里，在它的生命周期中，我们并没有使用原来的借用`r`，因此不会报错
    println!("{:?}", rr);
    // 再借用结束后，才去使用原来的借用`r`
    r.move_to(10, 10);
    println!("{:?}", r);
}
```

