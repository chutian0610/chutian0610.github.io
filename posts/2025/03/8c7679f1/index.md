# Rust-特征

如果不同的类型具有相同的行为，那么我们就可以定义一个特征，然后为这些类型实现该特征。定义特征是把一些方法组合在一起，目的是定义一个实现某些目标所必需的行为的集合。

例如，我们现在有文章 Post 和微博 Weibo 两种内容载体，而我们想对相应的内容进行总结，也就是无论是文章内容，还是微博内容，都可以在某个时间点进行总结，那么总结这个行为就是共享的，因此可以用特征来定义：

```rust
pub trait Summary {
    fn summarize(&self) -> String;
}
```

特征只定义行为看起来是什么样的，而不定义行为具体是怎么样的。因此我们需要为实现特征的类型，定义行为具体是怎么样的。

```rust
pub struct Post {
    pub title: String,
    pub author: String,
    pub content: String,
}
impl Summary for Post {
    fn summarize(&self) -> String {
        format!("文章{}, 作者是{}", self.title, self.author)
    }
}
```

<!--more-->

## 孤儿规则

上面我们将 Summary 定义成了 pub 公开的。这样，如果他人想要使用我们的 Summary 特征，则可以引入到他们的包中，然后再进行实现。

关于特征实现与定义的位置，有一条非常重要的原则：如果你想要为类型 A 实现特征 T，那么 A 或者 T 至少有一个是在当前作用域中定义的！ 

例如我们可以为上面的 Post 类型实现标准库中的 Display 特征，这是因为 Post 类型定义在当前的作用域中。同时，我们也可以在当前包中为 String 类型实现 Summary 特征，因为 Summary 定义在当前作用域中。但是你无法在当前作用域中，为 String 类型实现 Display 特征，因为它们俩都定义在标准库中，其定义所在的位置都不在当前作用域。

该规则被称为孤儿规则，可以确保其它人编写的代码不会破坏你的代码，也确保了你不会破坏其他的代码。

### 在外部类型上实现外部特征(newtype)

这里提供一个办法来绕过孤儿规则，那就是使用newtype 模式，简而言之：就是为一个元组结构体创建新类型。该元组结构体封装有一个字段，该字段就是希望实现特征的具体类型。该封装类型是本地的，因此我们可以为此类型实现外部的特征。

例如我们有一个动态数组类型： `Vec<T>`，它定义在标准库中，还有一个特征 Display，它也定义在标准库中，如果没有 newtype，我们是无法为 `Vec<T>` 实现 Display 的,此时可以使用 newType。

```rust
use std::fmt;
struct Wrapper(Vec<String>);
impl fmt::Display for Wrapper {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "[{}]", self.0.join(", "))
    }
}
fn main() {
    let w = Wrapper(vec![String::from("hello"), String::from("world")]);
    println!("w = {}", w);
}
```

## 默认实现

可以在特征中定义具有默认实现的方法，这样其它类型无需再实现该方法，或者也可以选择重载该方法：

```rust
pub trait Summary {
    fn summarize(&self) -> String {
        String::from("(Read more...)")
    }
}
/// 继承实现
impl Summary for Post {}

/// 重载方法
impl Summary for Weibo {
    fn summarize(&self) -> String {
        format!("{}发表了微博{}", self.username, self.content)
    }
}
```

## 同名方法

不同特征拥有同名的方法是很正常的事情，你没有任何办法阻止这一点；甚至除了特征上的同名方法外，在你的类型上，也有同名方法：

```rust
trait Pilot {
    fn fly(&self);
}

trait Wizard {
    fn fly(&self);
}
struct Human;
impl Pilot for Human {
    fn fly(&self) {
        println!("This is your captain speaking.");
    }
}
impl Wizard for Human {
    fn fly(&self) {
        println!("Up!");
    }
}
impl Human {
    fn fly(&self) {
        println!("*waving arms furiously*");
    }
}
```

### 优先调用类型上的方法

当调用 Human 实例的 fly 时，编译器默认调用该类型中定义的方法：

```rust
fn main() {
    let person = Human;
    person.fly();
}
/// *waving arms furiously*
```

### 调用特征上的方法

```rust
为了能够调用两个特征的方法，需要使用显式调用的语法：

fn main() {
    let person = Human;
    Pilot::fly(&person); // 调用Pilot特征上的方法
    Wizard::fly(&person); // 调用Wizard特征上的方法
    person.fly(); // 调用Human类型自身的方法
}
```

### 完全限定语法

完全限定语法是调用函数最为明确的方式：

```rust
trait Animal {
    fn baby_name() -> String;
}
struct Dog;
impl Dog {
    fn baby_name() -> String {
        String::from("Spot")
    }
}
impl Animal for Dog {
    fn baby_name() -> String {
        String::from("puppy")
    }
}
fn main() {
    println!("A baby dog is called a {}", <Dog as Animal>::baby_name());
}
```
在尖括号中，通过 as 关键字，我们向 Rust 编译器提供了类型注解，也就是 Animal 就是 Dog，而不是其他动物，因此最终会调用 `impl Animal for Dog` 中的方法。

完全限定语法定义为：

```rust
<Type as Trait>::function(receiver_if_method, next_arg, ...);
```

上面定义中，第一个参数是方法接收器 receiver(三种 self)，只有方法才拥有，例如关联函数就没有 receiver。

## 关联类型

关联类型是在特征定义的语句块中，申明一个自定义类型，这样就可以在特征的方法签名中使用该类型：

```rust
pub trait Iterator {
    type Item;
    // Self::Item 就用来指代该类型实现中定义的 Item 类型：
    fn next(&mut self) -> Option<Self::Item>;
}
impl Iterator for Counter {
    type Item = u32;
    fn next(&mut self) -> Option<Self::Item> {
        // --snip--
    }
}
```

通过关联类型，代码的可读性会比使用泛型更高。

## 函数参数的特征约束

定义一个函数，使用特征作为函数参数：

```rust
pub fn notify(item: &impl Summary) {
    println!("Breaking news! {}", item.summarize());
}
```
`impl Summary` 的意思是 实现了Summary特征 的 item 参数。这其实是个语法糖，依赖了特征约束(trait bound)语法。实际代码如下:

```rust
pub fn notify<T: Summary>(item: &T) {
    println!("Breaking news! {}", item.summarize());
}
```

对于复杂的场景，特征约束可以让我们拥有更大的灵活性和语法表现能力，例如一个函数接受两个 impl Summary 的参数：

```rust
pub fn notify(item1: &impl Summary, item2: &impl Summary) {}
```

如果函数两个参数是不同的类型，那么上面的方法很好，只要这两个类型都实现了 Summary 特征即可。但是如果我们想要强制函数的两个参数是同一类型呢？上面的语法就无法做到这种限制，此时我们只能使特征约束来实现：

```rust
pub fn notify<T: Summary>(item1: &T, item2: &T) {}
```

泛型类型 T 说明了 item1 和 item2 必须拥有同样的类型，同时 `T: Summary` 说明了 T 必须实现 Summary 特征。

### 多重约束

除了单个约束条件，我们还可以指定多个约束条件，例如除了让参数实现 Summary 特征外，还可以让参数实现 Display 特征以控制它的格式化输出：

```rust
pub fn notify(item: &(impl Summary + Display)) {}
pub fn notify<T: Summary + Display>(item: &T) {}
```

通过这两个特征，就可以使用 `item.summarize` 方法，以及通过 `println!("{}", item)` 来格式化输出 item。

### Where 约束

当特征约束变得很多时，函数的签名将变得很复杂：

```rust
fn some_function<T: Display + Clone, U: Clone + Debug>(t: &T, u: &U) -> i32 {}
```

通过 where能对其做一些形式上的改进:

```rust
fn some_function<T, U>(t: &T, u: &U) -> i32
    where T: Display + Clone,
          U: Clone + Debug
{}
```

### 使用特征约束有条件地实现方法或特征

特征约束，可以让我们在指定类型 + 指定特征的条件下去实现方法，例如：

```rust
use std::fmt::Display;
struct Pair<T> {
    x: T,
    y: T,
}
impl<T> Pair<T> {
    fn new(x: T, y: T) -> Self {
        Self { x, y }
    }
}
impl<T: Display + PartialOrd> Pair<T> {
    fn cmp_display(&self) {
        if self.x >= self.y {
            println!("The largest member is x = {}", self.x);
        } else {
            println!("The largest member is y = {}", self.y);
        }
    }
}
```

cmp_display 方法，并不是所有的 `Pair<T>` 结构体对象都可以拥有，只有 T 同时实现了 `Display + PartialOrd` 的 `Pair<T>` 才可以拥有此方法。 该函数可读性会更好，因为泛型参数、参数、返回值都在一起，可以快速的阅读，同时每个泛型参数的特征也在新的代码行中通过特征约束进行了约束。

也可以有条件地实现特征，例如，标准库为任何实现了 Display 特征的类型实现了 ToString 特征：

```rust
impl<T: Display> ToString for T {
    // --snip--
}
```
我们可以对任何实现了 Display 特征的类型调用由 ToString 定义的 to_string 方法。

## 函数返回的特征约束

可以通过 impl Trait 来说明一个函数返回了一个类型，该类型实现了某个特征：

```rust
fn returns_summarizable() -> impl Summary {
    Post {
        title: String::from("hello,world"),
        author: String::from("victorchu"),
        content: String::from(":-)")
    }
}
```
虽然我们知道这里是一个 Post 类型，但是对于 returns_summarizable 的调用者而言，他只知道返回了一个实现了 Summary 特征的对象，但是并不知道返回了一个 Post 类型。

但是这种返回值方式有一个很大的限制, 只能有一个具体的类型 ^[以下的代码就无法通过编译，因为它返回了两个不同的类型Post和Weibo。]:

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        Post {
            title: String::from(
                "Penguins win the Stanley Cup Championship!",
            ),
            author: String::from("Iceburgh"),
            content: String::from(
                "The Pittsburgh Penguins once again are the best \
                 hockey team in the NHL.",
            ),
        }
    } else {
        Weibo {
            username: String::from("horse_ebooks"),
            content: String::from(
                "of course, as you probably already know, people",
            ),
        }
    }
}
/// `if` and `else` have incompatible types
/// expected struct `Post`, found struct `Weibo`
```

## 特征对象

在上一节中有一段代码无法通过编译：

```rust
fn returns_summarizable(switch: bool) -> impl Summary {
    if switch {
        Post {
           // ...
        }
    } else {
        Weibo {
            // ...
        }
    }
}
```

其中 Post 和 Weibo 都实现了 Summary 特征，因此上面的函数试图通过返回 impl Summary 来返回这两个类型，但是编译器却无情地报错了，原因是 impl Trait 的返回值类型并不支持多种不同的类型返回，那如果我们想返回多种类型，该怎么办？

为了解决上面的所有问题，Rust 引入了一个概念 —— 特征对象。

```rust
/// 只要组件实现了 Draw 特征，就可以调用 draw 方法来进行渲染。
pub trait Draw {
    fn draw(&self);
}
pub struct Button {
    pub width: u32,
    pub height: u32,
    pub label: String,
}

impl Draw for Button {
    fn draw(&self) {
        // 绘制按钮的代码
    }
}

struct SelectBox {
    width: u32,
    height: u32,
    options: Vec<String>,
}

impl Draw for SelectBox {
    fn draw(&self) {
        // 绘制SelectBox的代码
    }
}
pub struct Screen {
    // 特征对象
    // dyn 关键字只用在特征对象的类型声明上，在创建时无需使用 dyn
    pub components: Vec<Box<dyn Draw>>,
    // pub components: Vec<& dyn Draw>,
}

fn main(){
    let screen = Screen {
        components: vec![
            Box::new(SelectBox {
                width: 75,
                height: 10,
                options: vec![
                    String::from("Yes"),
                    String::from("Maybe"),
                    String::from("No")
                ],
            }),
        ],
    };
    screen.run();
}
```

特征对象指向实现了 Draw 特征的类型的实例，也就是指向了 Button 或者 SelectBox 的实例，这种映射关系是存储在一张表中，可以在运行时通过特征对象找到具体调用的类型方法。

可以通过 `&dyn Draw` 借用^[使用借用时要注意生命周期问题]或者 `Box<dyn Draw>` 智能指针的方式来创建特征对象。

> 注意 dyn 不能单独作为特征对象的定义，例如下面的代码编译器会报错，原因是特征对象可以是任意实现了某个特征的类型，编译器在编译期不知道该类型的大小，不同的类型大小是不同的。而 `&dyn` 和 `Box<dyn>` 在编译期都是已知大小，所以可以用作特征对象的定义。

### 特征对象的动态分发

泛型是在编译期完成处理的：编译器会为每一个泛型参数对应的具体类型生成一份代码，这种方式是静态分发(static dispatch)，因为是在编译期完成的，对于运行期性能完全没有任何影响。

与静态分发相对应的是动态分发(dynamic dispatch)，在这种情况下，直到运行时，才能确定需要调用什么方法。之前代码中的关键字 dyn 正是在强调这一“动态”的特点。

当使用特征对象时，Rust 必须使用动态分发。编译器无法知晓所有可能用于特征对象代码的类型，所以它也不知道应该调用哪个类型的哪个方法实现。为此，Rust 在运行时使用特征对象中的指针来知晓需要调用哪个方法。动态分发也阻止编译器有选择的内联方法代码，这会相应的禁用一些优化。

![](layout.webp)

- 特征对象大小不固定：这是因为，对于特征 Draw，类型 Button 可以实现特征 Draw，类型 SelectBox 也可以实现特征 Draw，因此特征没有固定大小
- 几乎总是使用特征对象的引用方式，如 &dyn Draw、Box<dyn Draw>
    - 虽然特征对象没有固定大小，但它的引用类型的大小是固定的，它由两个指针组成（ptr 和 vptr），因此占用两个指针大小
    - 一个指针 ptr 指向实现了特征 Draw 的具体类型的实例，也就是当作特征 Draw 来用的类型的实例，比如类型 Button 的实例、类型 SelectBox 的实例
    - 另一个指针 vptr 指向一个虚表 vtable，vtable 中保存了类型 Button 或类型 SelectBox 的实例对于可以调用的实现于特征 Draw 的方法。当调用方法时，直接从 vtable 中找到方法并调用。

当类型 Button 实现了特征 Draw 时，类型 Button 的实例对象 btn 可以当作特征 Draw 的特征对象类型来使用，btn 中保存了作为特征对象的数据指针（指向类型 Button 的实例数据）和行为指针（指向 vtable）。

一定要注意，此时的 btn 是 Draw 的特征对象的实例，而不再是具体类型 Button 的实例，而且 btn 的 vtable 只包含了实现自特征 Draw 的那些方法（比如 draw），因此 btn 只能调用实现于特征 Draw 的 draw 方法，而不能调用类型 Button 本身实现的方法和类型 Button 实现于其他特征的方法。

### Self 与 self

在 Rust 中，有两个self，一个指代当前的实例对象，一个指代特征或者方法类型的别名：

```rust
trait Draw {
    fn draw(&self) -> Self;
}
#[derive(Clone)]
struct Button;
impl Draw for Button {
    fn draw(&self) -> Self {
        return self.clone()
    }
}
fn main() {
    let button = Button;
    let newb = button.draw();
}
```

上述代码中，self指代的就是当前的实例对象，也就是 button.draw() 中的 button 实例，Self 则指代的是 Button 类型。

### 特征对象的限制

不是所有特征都能拥有特征对象，只有对象安全的特征才行。当一个特征的所有方法都有如下属性时，它的对象才是安全的：

- 方法的返回类型不能是 Self
- 方法没有任何泛型参数

标准库中的 Clone 特征就不符合对象安全的要求：

```rust
pub trait Clone {
    fn clone(&self) -> Self;
}
```
因为它的clone方法，返回了 Self 类型，因此它是对象不安全的。

### Async Trait

如果在traits中编写async fn，例如`async fn foo(&self)`，trait以及impl块中语法糖会被解糖为：

```rust
trait Trait {
    async fn foo(&self);
}
impl Trait for TypeA {
    async fn foo(&self);
}
// ========== 解糖后 ============
trait Trait {
    // 匿名关联类型
    type Foo<'s>: Future<Output = ()> + 's;
    fn foo(&self) -> Self::Foo<'_>;
}
impl Trait for TypeA {
    // 匿名关联类型
    type Foo<'s> = impl Future<Output = ()> + 's;
    fn foo(&self) -> Self::Foo<'_> {
        async move { 
            // has some unique future type F_A
        } 
    }
}
```

使用异步函数的trait在dyn情况下并不安全，因为我们并不知道Future具体是什么类型；使用dyn时必须列出所有的关联类型的值。也就是说，如果要使用dyn，必须确定 Future 的实际类型：

```rust
// XXX是impl块定义的future类型
dyn for<'s> Trait<Foo<'s> = XXX>
```

这使dyn trait限制于某一个特定的impl块，而这与dyn trait的设计意图冲突：在使用dyn时，用户并不知道实际上的类型是什么，只知道类型实现了目标trait。出于这个原因，一个使用`#[async_trait]`的改进方式如下：

```rust
#[async_trait]
// to state whether Box<...> is send or not if desired: #[async_future(?Send)]
trait Trait {
    async fn foo(&self);
}
// 脱糖为
trait Trait {
    fn foo(&self) -> Box<dyn Future<Output = ()> + Send + '_>;
}
```

这样子做可以通过编译，缺点在于，哪怕不使用dyn trait，也会在堆上为Box分配空间，以及用户必须提早声明`Box<...>`是否实现了Sendtrait。这会带来不必要的麻烦，并且与Rust的设计意图冲突。

## 枚举

相较于使用trait的分发，rust 中 枚举也可以实现多态。

```rust
pub trait Draw {
    fn draw(&self);
}
pub struct Button {}

impl Draw for Button {
    fn draw(&self) {
       println!("this is a button");
    }
}

struct SelectBox{}

impl Draw for SelectBox {
    fn draw(&self) {
        println!("this is a SelectBox");
    }
}
enum UiObject {
    A(Button),
    B(SelectBox),
}

impl UiObject{
    fn draw(&self) {
    match self {
        UiObject::A(button) => &button.draw(),
        UiObject::B(select_box) => &select_box.draw(),
    };
}
}

fn main() {
    let objects = [
        UiObject::A(Button{}),
        UiObject::B(SelectBox{})
    ];

    for o in objects {
        o.draw()
    }
}
```

enum是封闭类型集，可以把没有任何关系的任意类型包裹成一个统一的单一类型。后续的任何变动，都需要改这个统一类型，以及基于这个 enum 的模式匹配等相关代码。而 impl trait 和 dyn trait 是开放类型集，只要对新的类型实现 trait，就可以传入使用了 impl trait 或 dyn trait 的函数，其函数签名不用变。

上述区别对于库的提供者非常重要，如果你提供了一个库，里面的多类型使用的 enum 包装，那么库的使用者没办法对你的 enum 进行扩展。因为一般来说，我们不鼓励去修改库里面的代码。而用 impl trait 或 dyn trait 就可以让接口具有可扩展性，用户只需要给他们的类型实现你的库提供的 trait，就可以代入库的接口使用了。

## 参考

- [1] [Async fn in dyn trait](https://rust-lang.github.io/async-fundamentals-initiative/explainer/async_fn_in_dyn_trait.html)

