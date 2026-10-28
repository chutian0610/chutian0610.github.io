# Java并发之CompletableFuture

Future接口可以构建异步应用，但依然有其局限性。它很难直接表述多个Future 结果之间的依赖性。实际开发中，我们经常需要达成以下目的：

* 将两个异步计算合并为一个——这两个异步计算之间相互独立，同时第二个又依赖于第一个的结果。
* 等待 Future 集合中的所有任务都完成。
* 仅等待 Future集合中最快结束的任务完成（有可能因为它们试图通过不同的方式计算同一个值），并返回它的结果。
* 通过编程方式完成一个Future任务的执行（即以手工设定异步操作结果的方式）。
* 应对 Future 的完成事件（即当 Future 的完成事件发生时会收到通知，并能使用 Future 计算的结果进行下一步的操作，不只是简单地阻塞等待操作的结果）

<!--more-->

## CompletableFuture

JDK1.8才新加入的一个实现类`CompletableFuture`，实现了`Future<T>`接口和 ` CompletionStage<T>` 接口。

## `CompletionStage<T>`

`CompletionStage<T>`是一个接口，从命名上看得知是一个完成的阶段，它里面的方法也标明是在某个运行阶段得到了结果之后要做的事情。


## 创建CompletableFuture对象

CompletableFuture.completedFuture是一个静态辅助方法，用来返回一个已经计算好的CompletableFuture。

```java
public static <U> CompletableFuture<U> completedFuture(U value);
```

而以下四个静态方法用来为一段异步执行的代码创建CompletableFuture对象：

```java
public static CompletableFuture<Void> 	runAsync(Runnable runnable)
public static CompletableFuture<Void> 	runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U> 	supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> 	supplyAsync(Supplier<U> supplier, Executor executor)
```

以Async结尾并且没有指定Executor的方法会使用ForkJoinPool.commonPool()作为它的线程池执行异步代码。runAsync方法也好理解，它以Runnable函数式接口类型为参数，所以CompletableFuture的计算结果为空。supplyAsync方法以`Supplier<U>`函数式接口类型为参数,CompletableFuture的计算结果类型为U。

## API

以下方法不以Async结尾，意味着Action使用相同的线程执行，而Async可能会使用其它的线程去执行(如果使用相同的线程池，也可能会被同一个线程选中执行)。

### 变换

```java
public <U> CompletionStage<U> thenApply(Function<? super T,? extends U> fn);
public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn);
public <U> CompletionStage<U> thenApplyAsync(Function<? super T,? extends U> fn,Executor executor);
```

上面的方法都是可以异步执行的，如果指定了线程池，会在指定的线程池中执行，如果没有指定，默认会在ForkJoinPool.commonPool()中执行，关键的入参只有一个Function，它是函数式接口，所以使用Lambda表示起来会更加优雅。它的入参是上一个阶段计算后的结果，返回值是经过转化后结果。

```java
public void thenApply() {
        String result = CompletableFuture.supplyAsync(() -> "hello").thenApply(s -> s + " world").join();
        System.out.println(result);
    }
// hello world
```

### 消费

```java
public CompletionStage<void> thenAccept(Consumer<? super T> action);
public CompletionStage<void> thenAcceptAsync(Consumer<? super T> action);
public CompletionStage<void> thenAcceptAsync(Consumer<? super T> action,Executor executor);
```

thenAccept是针对结果进行消耗，因为他的入参是Consumer，有入参无返回值。例如：

```java
public void thenAccept(){
       CompletableFuture.supplyAsync(() -> "hello").thenAccept(s -> System.out.println(s+" world"));
}
// hello world
```

### 对上一步的计算结果不关心，执行下一个操作

```java
public CompletionStage<void> thenRun(Runnable action);
public CompletionStage<void> thenRunAsync(Runnable action);
public CompletionStage<void> thenRunAsync(Runnable action,Executor executor);
```

thenRun它的入参是一个Runnable的实例，表示当得到上一步的结果时的操作。例如：

```java
public void thenRun(){
  CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(2000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "hello";
  }).thenRun(() -> System.out.println("hello world"));
  while (true){}
}
//hello world
```

### 结合两个CompletionStage的结果，进行转化后返回

```java
public <U,V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn,Executor executor);
```

它需要原来的处理返回值，并且other代表的CompletionStage也要返回值之后，利用这两个返回值，进行转换后返回指定类型的值。

```java
public void thenCombine() {
  String result = CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(2000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "hello";
  }).thenCombine(CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "world";
  }), (s1, s2) -> s1 + " " + s2).join();
  System.out.println(result);
}
```

### 结合两个CompletionStage的结果，进行消耗

```java
public <U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other,BiConsumer<? super T, ? super U> action,Executor executor);
```

它需要原来的处理返回值，并且other代表的CompletionStage也要返回值之后，利用这两个返回值，进行消耗。

```java
public void thenAcceptBoth() {
  CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(2000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "hello";
  }).thenAcceptBoth(CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "world";
  }), (s1, s2) -> System.out.println(s1 + " " + s2));
  while (true){}
}
//hello world
```

### 在两个CompletionStage都运行完执行

```java
public CompletionStage<Void> runAfterBoth(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

不关心这两个CompletionStage的结果，只关心这两个CompletionStage执行完毕，之后在进行操作（Runnable）。

```java
@Test
public void runAfterBoth(){
  CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(2000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "s1";
  }).runAfterBothAsync(CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "s2";
  }), () -> System.out.println("hello world"));
  while (true){}
}
// hello world
```

### 两个CompletionStage，谁计算的快，我就用那个CompletionStage的结果进行下一步的转化操作

```java
public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn,Executor executor);
```

我们现实开发场景中，总会碰到有两种渠道完成同一个事情，所以就可以调用这个方法，找一个最快的结果进行处理。

```java
@Test
public void applyToEither() {
  String result = CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "s1";
  }).applyToEither(CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(2000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "hello world";
  }), s -> s).join();
  System.out.println(result);
}
// hello world
```

### 两个CompletionStage，谁计算的快，我就用那个CompletionStage的结果进行下一步的消耗操作

```java
public CompletionStage<void> acceptEither(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action);
public CompletionStage<void> acceptEitherAsync(CompletionStage<? extends T> other,Consumer<? super T> action,Executor executor);
```

```java
@Test
public void acceptEither() {
  CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "s1";
  }).acceptEither(CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(2000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "hello world";
  }), System.out::println);
  while (true){}
}
//hello world
```

### 两个CompletionStage，任何一个完成了都会执行下一步的操作（Runnable）

```java
public CompletionStage<void> runAfterEither(CompletionStage<?> other,Runnable action);
public CompletionStage<void> runAfterEitherAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<void> runAfterEitherAsync(CompletionStage<?> other,Runnable action,Executor executor);
```

```java
@Test
public void runAfterEither() {
  CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "s1";
  }).runAfterEither(CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(2000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "s2";
  }), () -> System.out.println("hello world"));
  while (true) {
  }
}
// hello world
```

### 当运行时出现了异常，可以通过exceptionally进行补偿

```java
public CompletionStage<T> exceptionally(Function<Throwable, ? extends T> fn);
```

```java
@Test
public void exceptionally() {
  String result = CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      if (1 == 1) {
          throw new RuntimeException("测试一下异常情况");
      }
      return "s1";
  }).exceptionally(e -> {
      System.out.println(e.getMessage());
      return "hello world";
  }).join();
  System.out.println(result);
}
// java.lang.RuntimeException: 测试一下异常情况
// hello world
```

### 当运行完成时，对结果的记录

这里的完成时有两种情况:

* 一种是正常执行，返回值。
* 另外一种是遇到异常抛出造成程序的中断。

这里为什么要说成记录，因为这几个方法都会返回CompletableFuture，当Action执行完毕后它的结果返回原始的CompletableFuture的计算结果或者返回异常。所以不会对结果产生任何的作用。

```java
public CompletionStage<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);
public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action);
public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action,Executor executor);
```

```java
@Test
public void whenComplete() {
  String result = CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      if (1 == 1) {
          throw new RuntimeException("测试一下异常情况");
      }
      return "s1";
  }).whenComplete((s, t) -> {
      System.out.println(s);
      System.out.println(t.getMessage());
  }).exceptionally(e -> {
      System.out.println(e.getMessage());
      return "hello world";
  }).join();
  System.out.println(result);
}
// null
// java.lang.RuntimeException: 测试一下异常情况
// java.lang.RuntimeException: 测试一下异常情况
// hello world
```

这里也可以看出，如果使用了exceptionally，就会对最终的结果产生影响，它没有返回如果没有异常时的正确的值，这也就引出下面我们要介绍的handle。

### 运行完成时，对结果的处理。

这里的完成时有两种情况，一种是正常执行，返回值。另外一种是遇到异常抛出造成程序的中断。

```java
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```

```java
@Test
public void handle() {
  String result = CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      //出现异常
      if (1 == 1) {
          throw new RuntimeException("测试一下异常情况");
      }
      return "s1";
  }).handle((s, t) -> {
      if (t != null) {
          return "hello world";
      }
      return s;
  }).join();
  System.out.println(result);
}
// hello world

// 未出现异常时
@Test
public void handle() {
  String result = CompletableFuture.supplyAsync(() -> {
      try {
          Thread.sleep(3000);
      } catch (InterruptedException e) {
          e.printStackTrace();
      }
      return "s1";
  }).handle((s, t) -> {
      if (t != null) {
          return "hello world";
      }
      return s;
  }).join();
  System.out.println(result);
}
// s1
```

### CompletableFuture 自带多任务组合方法allOf和anyOf

* allOf是等待所有任务完成，构造后CompletableFuture完成
* anyOf是只要有一个任务完成，构造后CompletableFuture就完成

```java
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)
```
静态方法allOf能够创建一个返回值为空的CompletableFuture，当其所有的组件均完成时，它也会达到完成状态。（join方法通常用来返回某个CF的结果，为了查看allOf方法所组合起来的所有CF的结果，必须要对其进行单独地查询。）

```java
// 为了获取allof的返回值，使用如下方案
// 注意，泛型是List<T>
private static <T> CompletableFuture<List<T>> sequence(List<CompletableFuture<T>> futures) {
    CompletableFuture<Void> allDoneFuture =
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[futures.size()]));
        // allof 代表的future执行完后，futures也都全部完成,
        // 此处的join，不需要阻塞
    return allDoneFuture.thenApply(v ->
            futures.stream().
                    map(future -> future.join()).
                    collect(Collectors.<T>toList())
    );
}
```

### timeout

> 自 JDK9开始 CompletableFuture 支持 delays 和 timeouts。

原理是通过一个专用的延迟线程池Delayer(单线程的 ScheduledThreadPoolExecutor)调度一个“超时任务”，与原任务形成“竞态”关系，并使用线程安全的CAS操作来保证最终结果。

```java
// 在 timeout（单位在 java.util.concurrent.Timeunits units 中，比如 MILLISECONDS ）前以给定的 value 完成这个 CompletableFuture。返回这个 CompletableFuture。
public CompletableFuture<T> completeOnTimeout(T value, long timeout, TimeUnit unit)
// 如果没有在给定的 timeout 内完成，就以 java.util.concurrent.TimeoutException 完成这个 CompletableFuture，并返回这个 CompletableFuture。
public CompletableFuture<T> orTimeout(long timeout, TimeUnit unit)
```

