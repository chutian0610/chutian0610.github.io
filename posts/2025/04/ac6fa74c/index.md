# 发明服务特征

[Tower](https://github.com/tower-rs/tower)是一个模块化和可重用组件库，用于构建健壮的网络客户端和服务器。其核心是[Service](https://docs.rs/tower/latest/tower/trait.Service.html)特征。Service是一个异步函数，它接受请求并产生响应。然而，其设计的某些方面可能并不明显。与其解释目前Tower中存在的Service特征，让我们通过想象如果你从头开始，你会如何发明它来看看Service背后的动机。

<!--more-->

## 简易 Http 框架

想象一下，您正在Rust中构建一个小HTTP框架。该框架将允许用户通过提供接收请求和带有一些响应的回复的代码来实现HTTP服务器。您可能有这样的API：

```rust
// Create a server that listens on port 3000
let server = Server::new("127.0.0.1:3000").await?;

// Somehow run the user's application
server.run(the_users_application).await?;
```

问题是，`the_users_application`应该是什么？

最简单的可能是：

```rust
// HttpRequest和HttpResponse是我们框架提供的一些结构。
fn handle_request(request: HttpRequest) -> HttpResponse {
    // ...
}
```

有了这个，我们可以像这样实现 `Server::run`:

```rust
impl Server {
    async fn run<F>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> HttpResponse,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Call the handler provided by the user
                let response = handler(request);

                write_http_response(connection, response).await?;
            });
        }
    }
}
```

在上面的代码，我们有一个异步函数run，它接受一个闭包(接受HttpRequest并返回一个HttpResponse)。这意味着用户可以像下面这样使用我们的 Server:

```rust
fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
    } else {
        HttpResponse::not_found()
    }
}
// Run the server and handle requests using our `handle_request` function
server.run(handle_request).await?;
```

但是我们目前的设计有一个问题：不能异步处理请求。想象一下，我们的用户需要在处理请求时查询数据库或向其他服务器发送请求。目前，这需要在我们等待处理程序产生Response时阻塞。如果我们希望我们的服务器能够处理大量并发连接，我们需要能够在等待该请求异步完成时处理其他请求。可以通过让处理程序函数返回一个Future来解决这个问题:

```rust
impl Server {
    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
    where
        // `handler` now returns a generic type `Fut`...
        F: Fn(HttpRequest) -> Fut,
        // ...which is a `Future` whose `Output` is an `HttpResponse`
        Fut: Future<Output = HttpResponse>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Await the future returned by `handler`
                let response = handler(request).await;
                write_http_response(connection, response).await?;
            });
        }
    }
}
// Now an async function
async fn handle_request(request: HttpRequest) -> HttpResponse {
    if request.path() == "/" {
        HttpResponse::ok("Hello, World!")
    } else if request.path() == "/important-data" {
        // We can now do async stuff in here
        let some_data = fetch_data_from_database().await;
        make_response(some_data)
    } else {
        HttpResponse::not_found()
    }
}
// Running the server is the same
server.run(handle_request).await?;
```

请求处理现在可以调用其他异步函数。但是，仍然缺少一些东西。如果我们的处理程序遇到错误并且无法产生响应怎么办？让我们改为返回`Result`。

```rust
impl Server {
    async fn run<F, Fut>(self, handler: F) -> Result<(), Error>
    where
        F: Fn(HttpRequest) -> Fut,
        // The response future is now allowed to fail
        Fut: Future<Output = Result<HttpResponse, Error>>,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // Pattern match on the result of the response future
                match handler(request).await {
                    Ok(response) => write_http_response(connection, response).await?,
                    Err(error) => handle_error_somehow(error, connection),
                }
            });
        }
    }
}
```

## 增加更多的功能

现在，假设我们要确保所有请求及时完成或失败，而不是让客户端无限期地等待可能永远不会到达的响应。我们可以通过为每个请求添加超时来做到这一点。超时设置了允许handler使用的最长持续时间的限制。如果它在该时间内没有产生响应，则返回错误。这允许客户端重试该请求或向用户报告错误，而不是永远等待。

您的第一个想法可能是修改Server，以便它可以配置超时。然后它会在每次调用handler时应用该超时。然而，事实证明，您实际上可以在不修改Server的情况下添加超时。使用`tokio::time::timeout`，我们可以创建一个新的处理程序函数来调用我们以前的handle_request，但超时为30秒:

```rust
async fn handler_with_timeout(request: HttpRequest) -> Result<HttpResponse, Error> {
    let result = tokio::time::timeout(
        Duration::from_secs(30),
        handle_request(request)
    ).await;
    match result {
        Ok(Ok(response)) => Ok(response),
        Ok(Err(error)) => Err(error),
        Err(_timeout_elapsed) => Err(Error::timeout()),
    }
}
```

这提供了一个非常好的切入点。我们能够在不更改任何现有代码的情况下添加超时。让我们以这种方式再添加一个特性。假设我们正在构建一个JSON API，因此希望所有响应都有一个`Content-Type: application/json`标头。我们可以用类似的方式包装handler_with_timeout，并像这样修改响应:

```rust
async fn handler_with_timeout_and_content_type(
    request: HttpRequest,
) -> Result<HttpResponse, Error> {
    let mut response = handler_with_timeout(request).await?;
    response.set_header("Content-Type", "application/json");
    Ok(response)
}
```

设计可以以这种方式扩展的库非常强大，因为它允许用户通过分层新行为来扩展库的功能，而无需等待库维护人员添加对它的支持。它还使测试更容易，因为您可以将代码分解为小的隔离单元并为它们编写细粒度的测试，而无需担心所有其他部分。

然而，有一个问题。我们目前的设计允许我们通过将处理函数包装在一个实现该行为的新处理函数中，然后调用内部函数来编写新行为。这是可行的，但是如果我们想添加大量附加功能，它不能很好地扩展。想象一下，我们有许多handle_with_*函数，每个函数都添加了一点新的行为。必须对中间处理程序调用的链进行硬编码，这将变得很有挑战性。

如果我们能以某种方式组合这三个函数而不必硬编码确切的顺序，同时仍然能够像以前一样运行我们的处理程序, 那就太好了。

## Handler 特征

```rust
trait Handler {
    async fn call(&mut self, request: HttpRequest) -> Result<HttpResponse, Error>;
}
```

我们可以将所有的方法都变为实现 Handler 的具体类型，这样就可以方便调用 Handler 链。但是，Rust目前不支持异步trait方法，所以我们有两种选择：

- 让call返回一个装箱的未来:`Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>`.这就是 [async-trait](https://crates.io/crates/async-trait) 库的作用.
- 将关联type Future添加到Handler，以便用户选择自己的类型。

我们选择二，因为它是最灵活的。拥有具体Future类型的用户可以使用它而不需要Box的成本，而不关心的用户仍然可以使用`Pin<Box<…>>`。

```rust
trait Handler {
    type Future: Future<Output = Result<HttpResponse, Error>>;
    fn call(&mut self, request: HttpRequest) -> Self::Future;
}
```

> call使用&mut self是有用的，因为它允许处理程序在必要时更新其内部状态。

让我们将原始的handle_request函数转换为这个trait的实现:

```rust
struct RequestHandler;

impl Handler for RequestHandler {
    // We use `Pin<Box<...>>` here for simplicity, but could also define our
    // own `Future` type to avoid the overhead
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future {
        Box::pin(async move {
            // same implementation as we had before
            if request.path() == "/" {
                Ok(HttpResponse::ok("Hello, World!"))
            } else if request.path() == "/important-data" {
                let some_data = fetch_data_from_database().await?;
                Ok(make_response(some_data))
            } else {
                Ok(HttpResponse::not_found())
            }
        })
    }
}
```

对于超时，可以像下面这样定义一个通用的Timeout结构:

```rust
struct Timeout<T> {
    // T will be some type that implements `Handler`
    inner_handler: T,
    duration: Duration,
}
```

然后，我们可以为Handler实现`Timeout<T>`并委托给T的Handler实现:

```rust
impl<T> Handler for Timeout<T>
where
    T: Handler,
{
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future {
        Box::pin(async move {
            let result = tokio::time::timeout(
                self.duration,
                self.inner_handler.call(request),
            ).await;

            match result {
                Ok(Ok(response)) => Ok(response),
                Ok(Err(error)) => Err(error),
                Err(_timeout) => Err(Error::timeout()),
            }
        })
    }
}
```

重要代码是`self.inner_handler.call(request)`。这是我们委托给内部处理程序并让它做它的事情的地方。我们不知道它是什么，我们只知道它在完成时产生一个`Result<HttpResponse, Error>`。

但是这段代码不能完全编译。我们得到这样的错误：

```rust
error[E0759]: `self` has an anonymous lifetime `'_` but it needs to satisfy a `'static` lifetime requirement
   --> src/lib.rs:145:29
    |
144 |       fn call(&mut self, request: HttpRequest) -> Self::Future {
    |               --------- this data with an anonymous lifetime `'_`...
145 |           Box::pin(async move {
    |  _____________________________^
146 | |             let result = tokio::time::timeout(
147 | |                 self.duration,
148 | |                 self.inner_handler.call(request),
...   |
155 | |             }
156 | |         })
    | |_________^ ...is captured here, requiring it to live as long as `'static`
```

原因是我们捕获了一个`&mut self`并将其移动到异步块中。这意味着future的生命周期与`&mut self`的生命周期相关联。由于我们的程序需要在多线程运行，所以我们需要将`&mut self`转换为 owned self(通过 clone 可以实现)。

```rust
// this must be `Clone` for `Timeout<T>` to be `Clone`
#[derive(Clone)]
struct RequestHandler;

impl Handler for RequestHandler {
    // ...
}

#[derive(Clone)]
struct Timeout<T> {
    inner_handler: T,
    duration: Duration,
}

impl<T> Handler for Timeout<T>
where
    T: Handler + Clone,
{
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: HttpRequest) -> Self::Future {
        // Get an owned clone of `&mut self`
        let mut this = self.clone();

        Box::pin(async move {
            let result = tokio::time::timeout(
                this.duration,
                this.inner_handler.call(request),
            ).await;

            match result {
                Ok(Ok(response)) => Ok(response),
                Ok(Err(error)) => Err(error),
                Err(_timeout) => Err(Error::timeout()),
            }
        })
    }
}
```

在这种情况下克隆的开销很低，因为RequestHandler没有任何数据，`Timeout<T>`只添加Duration(即Copy)。又近了一步。我们现在得到了一个不同的错误：

```rust
error[E0310]: the parameter type `T` may not live long enough
   --> src/lib.rs:149:9
    |
140 |   impl<T> Handler for Timeout<T>
    |        - help: consider adding an explicit lifetime bound...: `T: 'static`
...
149 | /         Box::pin(async move {
150 | |             let result = tokio::time::timeout(
151 | |                 this.duration,
152 | |                 this.inner_handler.call(request),
...   |
159 | |             }
160 | |         })
    | |__________^ ...so that the type `impl Future` will meet its required lifetime bounds
```

现在的问题是`T`可以是任何类型。它甚至可以是一个包含引用的类型，比如`Vec<&'a str>`。我们需要响应future有一个'static生命周期，这样我们就可以更容易地传递它。

```rust
impl<T> Handler for Timeout<T>
where
    T: Handler + Clone + 'static,
{
    // ...
}
```

响应Future现在满足'static生命周期要求，因为它不包含引用（并且T包含的任何引用都是'static的）。

```rust
impl Server {
    async fn run<T>(self, mut handler: T) -> Result<(), Error>
    where
        T: Handler,
    {
        let listener = TcpListener::bind(self.addr).await?;

        loop {
            let mut connection = listener.accept().await?;
            let request = read_http_request(&mut connection).await?;

            task::spawn(async move {
                // have to call `Handler::call` here
                match handler.call(request).await {
                    Ok(response) => write_http_response(connection, response).await?,
                    Err(error) => handle_error_somehow(error, connection),
                }
            });
        }
    }
}

let handler = RequestHandler;
let handler = Timeout::new(handler, Duration::from_secs(30));

// `handler` has type `JsonContentType<Timeout<RequestHandler>>`
server.run(handler).await
```

## 使Handler更加灵活

Handler特性工作得很好，但目前它只支持我们的HttpRequest和HttpResponse类型。如果这些是通用的，那就太好了，这样用户就可以使用他们想要的任何类型。我们将请求设为trait的泛型类型参数，以便给定的服务可以接受许多不同类型的请求。这允许定义可用于不同协议的处理程序，而不仅仅是HTTP。我们将响应设为关联类型，因为对于任何给定的请求类型，只能有一种（关联的）响应类型：对应的调用返回的响应类型！

```rust
trait Handler<Request> {
    type Response;

    // Error should also be an associated type. No reason for that to be a
    // hardcoded type
    type Error;

    // Our future type from before, but now it's output must use
    // the associated `Response` and `Error` types
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    // `call` is unchanged, but note that `Request` here is our generic
    // `Request` type parameter and not the `HttpRequest` type we've used
    // until now
    fn call(&mut self, request: Request) -> Self::Future;
}
```

我们对RequestHandler的实现现在变成了:

```rust
impl Handler<HttpRequest> for RequestHandler {
    type Response = HttpResponse;
    type Error = Error;
    type Future = Pin<Box<dyn Future<Output = Result<HttpResponse, Error>>>>;

    fn call(&mut self, request: Request) -> Self::Future {
        // same as before
    }
}
```

`Timeout<T>`有点不同。由于它包装了一些其他Handler并添加了异步超时，它实际上并不关心请求或响应类型是什么，只要它包装的Handler使用相同的类型。Error类型有点不同。由于`tokio::time::timeout`返回`Result<T, tokio::time::error::Elapsed>`，我们必须能够转换一个`tokio::time::error::Elapsed`到内部Handler的错误类型。

```rust
// `Timeout` accepts any request of type `R` as long as `T`
// accepts the same type of request
impl<R, T> Handler<R> for Timeout<T>
where
    // The actual type of request must not contain
    // references. The compiler would tell us to add
    // this if we didn't
    R: 'static,
    // `T` must accept requests of type `R`
    T: Handler<R> + Clone + 'static,
    // We must be able to convert an `Elapsed` into
    // `T`'s error type
    T::Error: From<tokio::time::error::Elapsed>,
{
    // Our response type is the same as `T`'s, since we
    // don't have to modify it
    type Response = T::Response;

    // Error type is also the same
    type Error = T::Error;

    // Future must output a `Result` with the correct types
    type Future = Pin<Box<dyn Future<Output = Result<T::Response, T::Error>>>>;

    fn call(&mut self, request: R) -> Self::Future {
        let mut this = self.clone();

        Box::pin(async move {
            let result = tokio::time::timeout(
                this.duration,
                this.inner_handler.call(request),
            ).await;

            match result {
                Ok(Ok(response)) => Ok(response),
                Ok(Err(error)) => Err(error),
                Err(elapsed) => {
                    // Convert the error
                    Err(T::Error::from(elapsed))
                }
            }
        })
    }
}
```

最后，传递给`Server::run`的Handler必须使用HttpRequest和HttpResponse：

```rust
impl Server {
    async fn run<T>(self, mut handler: T) -> Result<(), Error>
    where
        T: Handler<HttpRequest, Response = HttpResponse>,
    {
        // ...
    }
}
// ========== Creating the server ============
let handler = RequestHandler;
let handler = Timeout::new(handler, Duration::from_secs(30));

server.run(handler).await
```

到目前为止，我们只讨论了服务器端的事情。但是，我们的Handler特性实际上也适合HTTP客户端。可以想象一个客户端Handler接受一些请求并将其异步发送给Internet上的某人。我们的Timeout包装器在这里也很有用。由于我们的Handler特征对于定义服务器和客户端都很有用，Handler可能不是一个合适的名称。客户端不处理请求，它将请求发送到服务器，然后服务器处理它。让我们改为调用我们的特征Service：

```rust
trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;
    fn call(&mut self, request: Request) -> Self::Future;
}
```

这实际上几乎是Tower中定义的Servicetrait。如果您能够一直理解到现在，您现在已经了解了Tower的大部分内容。除了Servicetrait之外，Tower还提供了几个实用程序，通过包装一些其他类型来实现Service，这些类型也实现了Service，就像我们对Timeout所做的那样。这些服务可以以类似于我们目前所做的方式组合。

Tower提供的一些示例服务：

- Timeout-这与我们构建的超时几乎相同。
- Retry-自动重试失败的请求。
- RateLimit-限制服务在一段时间内将收到的请求数。

## Backpressure

假设您想编写一个速率限制中间件来包装一个Service，并限制底层服务将接收的最大并发请求数。如果您有一些服务对它可以处理的负载量有硬上限，这将很有用。
根据我们当前的Service特性，我们并没有很好的方法来实现这样的东西。我们可以尝试:

```rust
impl<R, T> Service<R> for ConcurrencyLimit<T> {
    fn call(&mut self, request: R) -> Self::Future {
        // 1. Check a counter for the number of requests currently being
        //    processed.
        // 2. If there is capacity left send the request to `T`
        //    and increment the counter.
        // 3. If not somehow wait until capacity becomes available.
        // 4. When the response has been produced, decrement the counter.
    }
}
```

如果没有剩余容量，我们必须等待，并在容量可用时以某种方式得到通知。此外，我们必须在等待时将请求保存在内存中（也称为缓冲）。这意味着等待容量的请求越多，我们的程序将使用的内存就越多——如果产生的请求比我们的服务能够处理的要多，我们可能会运行内存溢出！

只有在我们确定服务有能力处理请求时才为请求分配空间会更健壮。否则，在我们等待服务准备就绪时，我们就有可能使用大量内存缓冲请求。

如果Service有这样的方法就好了：

```rust
trait Service<R> {
    async fn ready(&mut self);
}
```

ready是一个异步函数，当服务有足够的能力接收一个新请求时完成。然后，我们要求用户在执行`service.call(request).await`之前首先调用`service.ready().await`。

将"调用服务"与"预留容量"分开也解锁了新的用例，例如能够维护一组我们在后台保持最新的"就绪服务"，这样当请求到达时，我们已经有一个就绪服务可以将其发送到，而不必首先等待它准备好。通过这种设计，ConcurrencyLimit可以跟踪内部ready的容量，并且不允许用户call，直到有足够的容量。

不关心容量的Service可以立即从ready返回，或者如果它们包装了一些内部Service，它们可以委托给它的ready方法。然而，我们仍然不能在特征中定义异步函数。我们可以向Service添加另一个关联类型，称为ReadyFuture，但是必须返回一个Future会给我们带来以前遇到的相同的生命周期问题。如果有某种方法可以解决这个问题，那就太好了。

相反，我们可以从Future trait中获得一些灵感，并定义一个名为poll_ready的方法：

```rust
use std::task::{Context, Poll};

trait Service<R> {
    fn poll_ready(&mut self, cx: &mut Context<'_>) -> Poll<()>;
}
```

这意味着如果服务容量不足，`poll_ready`将返回`Poll::Pending`，并在容量可用时使用Context中的waker通知调用者。此时，可以再次调用`poll_ready`，如果它返回`Poll::Ready(())`，则保留容量并调用call。

请注意，从技术上讲，没有什么可以阻止用户在没有首先确保服务准备好的情况下调用call。但是，这样做被认为违反了Service API约定，并且如果在未准备好的服务上调用call，则允许实现panic！

poll_ready不返回Future也意味着我们能够快速检查服务是否准备就绪，而无需被迫等待它准备就绪。如果我们调用poll_ready并返回Poll::Pending，我们可以简单地决定做其他事情而不是等待。除其他外，这允许您构建负载均衡器，通过服务返回的频率来估计服务的负载Poll::Pending，并向负载最少的服务发送请求。当容量可用时，使用futures::future::poll_fn（或tower::ServiceExt::ready）仍然有可能得到一个Future。

这种服务与调用者就其容量进行通信的概念称为“背压传播”。您可以将其视为服务对调用者的反击，并告诉他们如果产生的请求太快就放慢速度。基本思想是，您不应该向没有能力处理请求的服务发送请求。相反，您应该等待（缓冲）、放弃请求（负载卸载）或以其他方式处理容量不足。您可以在[这里](https://medium.com/@jayphelps/backpressure-explained-the-flow-of-data-through-software-2350b3e77ce7)和[这里](https://aws.amazon.com/builders-library/using-load-shedding-to-avoid-overload/)了解更多关于背压的一般概念。

最后，保留容量时也可能发生一些错误，因此poll_ready可能应该返回`Poll<Result<(), Self::Error>>`。通过此更改，我们现在已经到达了完整的`tower::Service`特征：

```rust
pub trait Service<Request> {
    type Response;
    type Error;
    type Future: Future<Output = Result<Self::Response, Self::Error>>;

    fn poll_ready(
        &mut self,
        cx: &mut Context<'_>,
    ) -> Poll<Result<(), Self::Error>>;

    fn call(&mut self, req: Request) -> Self::Future;
}
```

许多中间件不添加自己的反压，只是委托给包装服务的poll_ready实现。然而，中间件中的反压确实支持一些有趣的用例，例如各种速率限制、负载平衡和自动缩放。由于您永远不会确切知道Service可能由哪个中间件组成，因此不要忘记poll_ready很重要。

```rust
调用服务的最常见方法是：

use tower::{
    Service,
    // for the `ready` method
    ServiceExt,
};

let response = service
    // wait for the service to have capacity
    .ready().await?
    // send the request
    .call(request).await?;
```

