# 异步编程模型:Promises vs Futures

本文将介绍两种异步编程模型: Promise 和 Future。在计算机科学中，future、promise，是在某些并发编程语言中，指称用于同步程序执行的一种构造。由于某些计算尚未结束，故而需要一个对象来代理这个未知的结果。Future 和 Promise 的设计理念整体上非常相似，但是在不同的语言和框架实现中又存在一定的区别，对此，这里我们基于最广泛的定义进行介绍。

- Future：表示异步任务的返回值，表示一个未来值(只读)的占位符，即未来值的消费者。
- Promise：表示异步任务的执行过程，表示一个可写的单赋值容器，即未来值的生产者。

在同时包含 Future 和 Promise 的实现中，一般 Promise 对象会有一个关联的 Future 对象。当 Promise 创建时，Future 对象会自动实例化。当异步任务执行完毕，Promise 在内部设置结果，从而将值绑定至 Future 的占位符中。Future 则提供读取方法。将异步操作分成 Future 和 Promise 两个部分的主要原因是 为了实现读写分离，对外部调用者只读，对内部实现者只写。

<!--more-->

## Scala Promises vs Futures

在 Scala 中，Future 和 Promise 可作为同一个异步操作的两个部分。

- Future 作为一个可提供只读占位符，用于存储未来值的对象。
- Promise 作为一个实现一个 Future，并支持可写操作的单一赋值容器。

```scala
import scala.concurrent.{ Future, Promise }
import scala.concurrent.ExecutionContext.Implicits.global

val p = Promise[T]()
val f = p.future

val producer = Future {
  val r = produceSomething()
  p success r
  continueDoingSomethingUnrelated()
}

val consumer = Future {
  startDoingSomething()
  f onSuccess {
    case r => doSomethingWithResult()
  }
}
```

在这里，我们创建了一个promise并利用它的 future 方法获得由它实现的 Future。然后，我们开始了两种异步计算。第一种做了某些计算，结果值存放在r中，通过执行promise p，这个值被用来完成future对象 f。第二种做了某些计算，然后读取实现了 future f 的计算结果值 r。需要注意的是，在 producer 完成执行 continueDoingSomethingUnrelated() 方法这个任务之前，消费者可以获得这个结果值。

> promises 具有单赋值语义。因此，它们仅能被完成一次。在一个已经计算完成的 promise 或者 failed 的promise上调用 success 方法将会抛出一个 IllegalStateException 异常。

### Future Callback

在许多future的实现中，一旦future的client对future的结果感兴趣，它不得不阻塞它自己的计算直到future完成——然后才能使用future的值继续它自己的计算。 但是从性能的角度来看更好的办法是一种完全非阻塞的方法，即在future中注册一个回调。 future 完成后这个回调称为异步回调。如果当注册回调时 future 已经完成，则回调可能是异步执行的，或在相同的线程中循序执行。

注册回调最通常的形式是使用 OnComplete 方法，即创建一个`Try[T] => U`类型的回调函数。如果future成功完成，回调则会应用到 `Success[T]` 类型的值中，否则应用到 `Failure[T]` 类型的值中。

```scala
import scala.util.{Success, Failure}

val f: Future[List[String]] = Future {
  session.getRecentPosts()
}

f.onComplete {
  case Success(posts) => for post <- posts do println(post)
  case Failure(t) => println("An error has occurred: " + t.getMessage)
}
```

因为回调需要 future 的值是可用的，所有回调只能在 future 完成之后被调用。 然而，不能保证回调在完成 future 的线程或创建回调的线程中被调用。 反而， 回调会在 future 对象完成之后的一些线程和一段时间内执行。所以我们说回调最终会被执行。

此外，回调(callback)执行的顺序不是预先定义的，甚至在相同的应用程序中回调的执行顺序也不尽相同。 事实上，回调也许不是一个接一个连续的调用，但是可能会在同一时间同时执行。 

### Combinator

通过使用 promises，futures 的 onComplete 方法和 future 的构造方法，你能够实现函数式组合组合器(compition combinators)。 让我们来假设一下你想实现一个新的组合器 first，该组合器使用两个future f 和 g，然后生产出第三个 future，该future能够用 f 或者 g 来完成（看哪一个先到），但前提是它能够成功。

```scala
def first[T](f: Future[T], g: Future[T]): Future[T] =
  val p = Promise[T]

  f.foreach { x =>
    p.trySuccess(x)
  }

  g.foreach { x =>
    p.trySuccess(x)
  }

  p.future
```
> 注意，在这种实现方式中，如果 f 和 g 都不成功，那么 first(f, g) 将不会完成（即返回一个值或者返回一个异常）。

## Java

Java 1.5 提供了 Future 和 FutureTask，其中 Future 是一个接口，FutureTask 是一种实现，它们提供了一种相对标准的 Future 实现。其通过 Runnable 和 Callable 进行实例化，有一个无参构造器，Future 和 FutureTask 支持外部只读，FutureTask 的 set 方法是 protected，未来值只能由内部进行设置。如下所示，为基于 FutureTask 的一个应用示例。

```java
public class Test {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        executor.submit(futureTask);
        executor.shutdown();
        
        try {
            System.out.println("task result: "+ futureTask.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

Java 8 提供了 CompletableFuture，其本质上是一种 Promise 的实现。按照我们之前的定义，Future 是只读的，Promise 是可写的，而 CompletableFuture 提供了可由外部调用的状态更新方法，因此可以将其归类为 Promise。另一方面，CompletableFuture 又实现了 Future 的读取方法 get。整体上，CompletableFuture 混合了 Future 和 Promise 的能力。如下所示，为 CompletableFuture 的一个应用示例。

```java
Supplier<Integer> momsPurse = ()-> {
    try {
        Thread.sleep(1000);//mom is busy
    } catch (InterruptedException e) {
        ;
    }
    return 100;
};

ExecutorService ex = Executors.newFixedThreadPool(10);

CompletableFuture<Integer> promise =  
CompletableFuture.supplyAsync(momsPurse, ex);
promise.thenAccept(u->System.out.println("Thank you mom for $" + u ));
promise.complete(10); 
```

## 实现区别

其他很多编程语言中，并不同时包含 Future 和 Promise 两种结构，比如：Dart 只包含 Future，JavaScript 只包含 Promise，甚至有些编程语言混淆了 Future 和 Promise 的原始区别。

在独立实现中，Future 和 Promise 各自都有着相对比较统一的表示形式，在实现方面的差异也相对比较一致，主要包括以下几个方面区别：

- 状态表示
- 状态更新
- 返回机制

### 状态表示

在状态表示方面，Future 只有两种状态：

- uncomplete：表示未完成状态，即未来值还未计算出来。
- completed：表示已完成状态，即未来值已经计算出来。当然计算结果可以分为值或错误两种情况。

对于 Promise，一般使用三种状态进行表示：

- pending：待定状态，即 Promise 的初始状态。
- fulfilled：满足状态，表示任务执行成功。
- rejected：拒绝状态，表示任务执行失败。

无论是 Future 还是 Promise，状态转移的过程都是不可逆的。

### 状态更新

在状态更新方面，Future 的状态由 内部进行自动管理。当异步任务执行完成或抛出错误时，其状态将隐式地自动从 uncomplete 状态更新为 completed 状态。
对于 Promise，其状态由 外部进行手动管理。通常由开发者根据控制流逻辑，执行特定的状态更新方法显式地从 pending 状态更新为 fulfilled 或 rejected 状态。

### 返回机制

在返回机制方面，Future 以传统的 return 方式返回结果。通常，其返回正如普通的方法一样，通过 return 完成。而 Promise 通常将结果作为闭包参数进行传递，并执行闭包从而实现返回。如下所示为 JavaScript 中 Promise 的返回机制示例，resolve 是一个只接受成功值的闭包，其参数为 Image 类型；reject 是一个只接受错误值的闭包，其参数为 Error 类型。

```js
function loadImageAsync(url) {
  return new Promise(function(resolve, reject) {
    const image = new Image();
    image.onload = function() {
      resolve(image);
    };
    image.onerror = function() {
      reject(new Error('Could not load image at ' + url));
    };
    image.src = url;
  });
}
```

