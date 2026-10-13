# Rust Struct 字段自引用问题

先来看一段 Java 代码，Application中有version和 logger。logger 依赖了 Version。

```java
public class Application
{
    public Version version;
    public Logger logger;
    public Application() {
        version = new Version(1);
        logger =  new Logger(version);
    }

    public static void main(String[] args)
    {
        Application application = new Application();
        application.logger.log("Hello World!");
        // console output:
        // [version 1] Hello World!
    }
    class Version {
        public int ver;
        public Version (int ver){
            this.ver = ver;
        }
        public String toString() {
            return String.format("version %d",ver);
        }
    }
    class Logger {
        public Version version;
        public Logger (Version version){
            this.version = version;
        }
        void log(String msg) {
            System.out.println(String.format("[%s] %s",version,msg));
        }
    }
}
```

**那么问题来了，如何在 Rust 中实现相同的代码？**

<!--more-->

## Clone

由于所有权问题，Application 的构造方法需要消耗一个Version，同时Logger的构造也需要消耗一份 Version。所以比较简单的方式就是 Version 实现Clone trait。

但是如果 Version 是个复杂的 Struct，那么 Clone 方法的性能消耗会很高，这并不是一个好的方案。

```rust
#[derive(Clone)]
struct Version{
    ver: i32,
}
impl Version {
    fn new() -> Self {
        Self { ver:1 }
    }
}

struct Logger {
    version: Version,
}
impl Logger {
    fn new(version: Version) -> Self {
        Self { version }
    }
}

struct Application {
    ver: Version,
    logger: Logger,
}
impl Application {
    pub fn new() -> Self {

        let ver = Version::new();
        let logger = Logger::new(ver.clone());
        Self { ver, logger }
    }
}
```

### ~~borrow~~

 ~~为了避免 Clone 的性能消耗，我们考虑使用logger 中使用 version 的借用。~~

```rust
struct Version {
    ver: i32,
}
impl Version {
    fn new() -> Self {
        Self { ver: 1 }
    }
}
struct Logger<'a> {
    version: &'a Version,
}
impl<'a> Logger<'a> {
    fn new(version: &'a Version) -> Self {
        Self { version }
    }
}

struct Application<'a> {
    ver: Version,
    logger: Logger<'a>,
}
impl<'a> Application<'a> {
    pub fn new() -> Self {
        let version = Version::new();
        let logger = Logger::new(&version);
        Self { version, logger }
    }
}
```

 ~~但是这种情况下会遇到生命周期问题~~

```rust
error[E0515]: cannot return value referencing local variable `ver`
  --> src/main.rs:26:9
   |
25 |         let logger = Logger::new(&ver);
   |                                  ---- `ver` is borrowed here
26 |         Self { ver, logger }
   |         ^^^^^^^^^^^^^^^^^^^^ returns a value referencing data owned by the current function

error[E0505]: cannot move out of `ver` because it is borrowed
  --> src/main.rs:26:16
   |
22 | impl<'a> Application<'a> {
   |      -- lifetime `'a` defined here
23 |     pub fn new() -> Self {
24 |         let ver = Version::new();
   |             --- binding `ver` declared here
25 |         let logger = Logger::new(&ver);
   |                                  ---- borrow of `ver` occurs here
26 |         Self { ver, logger }
   |         -------^^^----------
   |         |      |
   |         |      move out of `ver` occurs here
   |         returning this value requires that `ver` is borrowed for `'a`
```

 ~~很显然version的生命周期和结构体一致，而Logger中&ver的生命周期是'a,超过了结构体的生命周期。只能将 version 的所有权移出结构体~~

```rust
struct Application<'a> {
    ver: &'a Version,
    logger: Logger<'a>,
}
impl<'a> Application<'a> {
    pub fn new(ver: &'a Version) -> Self {
        let logger = Logger::new(&ver);
        Self { ver, logger }
    }
}
```
### Rc or Arc

Rc和 Arc 表示共享的智能指针。可以用于值被共享的场景。

```rust
use std::rc::Rc;
struct Version{
    ver: i32,
}
impl Version {
    fn new() -> Self {
        Self { ver:1 }
    }
}
struct Logger {
    version: Rc<Version>,
}
impl Logger {
    fn new(version: Rc<Version>) -> Self {
        Self { version }
    }
}
struct Application {
    ver: Rc<Version>,
    logger: Rc<Logger>,
}
impl Application {
    pub fn new() -> Self {
        let ver = Rc::new(Version::new());
        let logger = Rc::new(Logger::new(Rc::clone(&ver)));
        Self { ver,logger }
    }
}
```

### Ouroboros

[Ouroboros](https://github.com/someguynamedjosh/ouroboros)是一个自引用结构体生成器。

```rust
struct Version {
    ver: i32,
}
impl Version {
    fn new() -> Self {
        Self { ver: 1 }
    }
}
struct Logger<'a> {
    version: &'a Version,
}
impl<'a> Logger<'a> {
    fn new(version: &'a Version) -> Self {
        Self { version }
    }
}

use ouroboros::self_referencing;

#[self_referencing]
struct ApplicationFields {
    ver: Version,

    #[borrows(ver)]
    #[covariant]
    logger: Logger<'this>,
}
// wrapper so we can define our own "new" function, using
// the ouroboros-generated one internally
struct Application(ApplicationFields);

impl Application {
    fn new() -> Self {
        Self(
            // let's use the more fancy …Builder API instead of
            // `ApplicationFields::new` (both are essentially equivalent though)
            ApplicationFieldsBuilder {
                ver: Version::new(),
                logger_builder: |version| Logger::new(version),
            }
            .build(),
        )
    }
}
```

