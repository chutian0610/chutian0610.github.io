# Guice简介

Guice是一个轻量级依赖注入框架。关于什么是依赖注入可以查看以前的blog，这里就不赘述了。

<!--more-->

## QuickStart

我们先来看下Guice的快速使用。`@Inject`注释的Java类构造函数可以由Guice调用，构造函数的参数将由 Guice 创建和提供。

```java
public class Greeter {
    private final String message;
    private final int count;

    /**
     *   Greeter declares that it needs a string message and an integer
     *   representing the number of time the message to be printed.
     *   The @Inject annotation marks this constructor as eligible to be used by
     *   Guice.
     */
    @Inject
    Greeter(@Message String message,
            // 此处的@Message标识注入的的String实例的key是Message.class
            @Count int count) {
        this.message = message;
        this.count = count;
    }

    void sayHello() {
        for (int i=0; i < count; i++) {
            System.out.println(message);
        }
    }
}
```

Guice Moudle 允许应用程序指定如何满足其依赖关系。

```java
@Qualifier
@Retention(RUNTIME)
public @interface Message {}

@Qualifier
@Retention(RUNTIME)
public @interface Count {}

// module提供了Greeter中依赖的message和count
public class GreeterModule extends AbstractModule {

    @Provides
    @Count
    static Integer provideCount(){
        return 3;
    }

    @Provides
    @Message
    static String provideMessage() {
        return "hello world";
    }
}
```

最后我们需要为Guice模块创建一个注入器。当请求给定类型的实例时，注入器找出要构造的对象，解析它们的依赖关系，并将所有实例连接起来。

```java
public static void main(String[] args) {
  /*
   * Guice.createInjector() takes one or more modules, and returns a new Injector
   * instance. Most applications will call this method exactly once, in their
   * main() method.
   */
  Injector injector = Guice.createInjector(new GreeterModule());

  /*
   * Now that we've got the injector, we can build objects.
   */
  Greeter greeter = injector.getInstance(Greeter.class);

  // Prints "hello world" 3 times to the console.
  greeter.sayHello();
}
```

> 完整代码见[github]https://github.com/chutian0610/code-lab/tree/main/demos/guice-quickstart)

## Guice 使用

简单来说，Guice可以帮助应用程序创建和查找对象(依赖项)。Guice可以视作一个映射集合(Guice Map)，其中的每个元素都有两个部分:

- Guice键，依赖项的Key，用于从Guice中获取依赖项实例
- Provider，用于创建依赖项实例的工具

### Key

`com.google.inject.Key`是Guice中标识依赖项的Key。

最简单的形式是通过java中的类型表示Key.

```java
表示一个String类型依赖项的key
Key<String> dbKey = Key.get(String.class)
```

对于相同类型依赖项，Guice使用绑定注解来区分，例如:

```java
final class MultilingualGreeter {
  private String englishGreeting;
  private String spanishGreeting;

  @Inject
  MultilingualGreeter(
      @English String englishGreeting, @Spanish String spanishGreeting) {
    this.englishGreeting = englishGreeting;
    this.spanishGreeting = spanishGreeting;
  }
}
```

通过注入器构建MultilingualGreeter实例的过程，相当于执行以下操作:

```java
MultilingualGreeter greeter = injector.getInstance(MultilingualGreeter.class)
// 相当于
String english = injector.getInstance(Key.get(String.class, English.class));
String spanish = injector.getInstance(Key.get(String.class, Spanish.class));
MultilingualGreeter greeter = new MultilingualGreeter(english, spanish);
```

### Provider

Guice使用`com.google.inject.Provider`来表示能够创建指定类型对象的工厂。

```java
public interface Provider<T> extends javax.inject.Provider<T> {
  @Override
  T get();
}
```

每个实现类可以构造或者从缓存中返回预先构建的实例。但是实际上大多数应用程序不直接实现接口，它们通过Guice Module配置。例如:

```java
//Guice moudle 创建两个 provider
public class GreeterModule extends AbstractModule {

    @Provides
    @Count
    static Integer provideCount(){
        return 3;
    }

    @Provides
    @Message
    static String provideMessage() {
        return "hello world";
    }
}
```

### 配置映射

使用Guice有两个部分：

- 配置：将内容添加到`Guice 映射`中。
- 注入：应用程序从`Guice映射`创建和检索对象。

Guice映射是使用Guice模块配置的。有两种方法：

- 添加方法注释，例如`@Provides`
- 使用 Guice 域特定语言 （DSL）。

从概念上讲，这些 DSL API只是提供操作 Guice 映射的方法。他们所做的操作非常简单。这里有一些例子:

|DSL语法|映射操作|描述|
|:---|:---|:---|
|`bind(key).toInstance(value)`|`map.put(key,() -> value)`|实例绑定|
|`bind(key).toProvider(provider)`|`map.put(key,provider)`|provider绑定|
|`bind(key).to(another)`|`map.put(key,map.get(anotherKey))`|链接绑定|
|`@Provides Foo provideFoo(){...}`|`map.put(key.get(Foo.class),module::provideFoo)`|提供程序绑定|

Guice使用`@Inject`注解描述注入，`@Inject`可以作用在方法，字段和构造器上。下面是构造器的注入例子:

```java
class Foo {
  private Database database;

  @Inject
  Foo(Database database) {  // We need a database, from somewhere
    this.database = database;
  }
}
@Provides
Database provideDatabase(
    // We need the @DatabasePath String before we can construct a Database
    @DatabasePath String databasePath) {
  return new Database(databasePath);
}
```
当注入具有自身依赖关系的事物时，Guice递归注入依赖项。可以想象，为了注入如上所示的Foo实例，Guice 创建了如下所示的Provider实现:

```java
class FooProvider implements Provider<Foo> {
  @Override
  public Foo get() {
    Provider<Database> databaseProvider = guiceMap.get(Key.get(Database.class));
    Database database = databaseProvider.get();
    return new Foo(database);
  }
}

class ProvideDatabaseProvider implements Provider<Database> {
  @Override
  public Database get() {
    Provider<String> databasePathProvider =
        guiceMap.get(Key.get(String.class, DatabasePath.class));
    String databasePath = databasePathProvider.get();
    return module.provideDatabase(databasePath);
  }
}
```

依赖关系形成有向图，注入通过执行深度优先来工作(从您想要的对象遍历图形，遍历其所有依赖项)。Guice Injector对象表示整个依赖关系图。要创建一个 Injector，Guice 需要验证整个图是否正常工作。不可能存在任何需要依赖但未提供依赖关系的"悬空"节点。如果图因任何原因无效，Guice 抛出一个 CreationException 描述问题。

## Scope

默认情况下，Guice每次获取值时会返回一个新实例，这个行为是通过Scope来配置的。Scope允许应用级别，session级别或请求级别的生命周期实例重用。

### 内置Scope

- Singleton: Guice自带一个默认内置Scope,Singleton Scope会在应用程序生命周期级别重用实例。支持通过`javax.inject.Singleton`或`com.google.inject.Singleton`标识。
- RequestScoped: Guice的Servlet扩展包含了web程序中常用的其他作用域，例如`@RequestScoped`是请求级别的生命周期实例重用。

### 使用Scope

Guice使用实现类上的注解来标识Scope。例如使用`@Singleton`来表示单例:

```java
@Singleton
public class InMemoryTransactionLog implements TransactionLog{
  ... 
}
```

`@Singleton`注解也可以作用在`@Provides`方法上:

```java
@Provides @Singleton
TransactionLog provideTransactionLog() {
  ...
}
```

也可以使用Guice DSL 中的bind方法来配置作用域。

```java
bind(TransactionLog.class)
.to(InMemoryTransactionLog.class)
// in 方法既可以使用Scope注解类，例如 Singleton.class
// 也可以使用Scope实例，例如 Scopes.SINGLETON,Scopes.NO_SCOPE
.in(Singleton.class);
```

如果在类型上申明的scope和通过`bind()`声明的scope相互冲突，Guice会使用`bind()`方法中声明的scope。如果类型上声明了不期望的scope，可以将该类型绑定到`Scopes.NO_SCOPE`。

`bind()`表达式中的scope作用于Source，而不是Target。

```java
bind(Bar.class).to(Applebees.class).in(Singleton.class);
bind(Grill.class).to(Applebees.class).in(Singleton.class);
```

对于上面的case，应用程序会有2个Applebees实例，分别视为Bar的实现和Grill的实现。所以，如果要确保一个类型是单例，可以在当前类上直接使用`@Singleton`注解。或者增加一个绑定关系:

```java
//这个绑定关系会导致上面的两个.in(Singleton.class)绑定无效
bind(Applebees.class).in(Singleton.class);
```

### 饿汉单例

Guice可以将单例定义为饿汉单例，以使单例被提前创建。

```java
bind(TransactionLog.class)
  .to(InMemoryTransactionLog.class)
  .asEagerSingleton();
```

饿汉单例能在快速暴露初始化问题，而默认的懒汉单例可以实现快速的“编辑-编译-运行”开发循环。

|PRODUCTION|DEVELOPMENT|
|:---|:---|
|`.asEagerSingleton()`|eager|eager|
|`.in(Singleton.class)`|eager|lazy|
|`.in(Scopes.SINGLETON)`|eager|lazy|
|`@Singleton`|eager|lazy|

> 注意，`@Singleton`生效的先决条件是被注释的类型已经在模块中配置。

### 选择Scope

如果对象是有状态的，那么它的Scope是很明显的。应用级别的是`@Singleton`，请求级别的是`@RequestScoped`。如果对象是无状态的且创建成本低，Scope是并不需要的，不给绑定设置Scope，Guice会每次创建新的实例。

单例在Java应用程序中很流行，但它们没有提供太多价值，特别是在涉及依赖注入时。尽管单例保存了对象的创建(以及后来的垃圾回收)，但初始化单例需要同步;获取初始化单例的句柄需要volatile修饰(注:此处指的应该是双重校验锁)。单例的用处:

- 有状态对象，例如配置或计数器
- 构造或查找代价昂贵的对象
- 占用资源的对象，例如数据库连接池。

### Scope和并发

被`@Singleton`和`@SessionScoped`注释的类必须是线程安全的。同时注入到这些类的所有依赖也必须是线程安全的。为了保证线程安全，尽可能使用构造函数注入来创建不可变对象（类的所有字段都是 final，并由单个带注释的构造函数初始化）。

`@RequestScoped`对象不需要线程安全，对于`@Singleton`和`@SessionScoped`对象，依赖于`@RequestScoped`对象会报错，**当宽Scope需要从窄Scope中获取对象时，可以注入一个对象的Provider**。

### 在测试中使用NO_SCOPE

如果你在Guice中使用Scope，特别是自定义Scope，但是在测试中并不关心Scope，可以使用Guice的NO_SCOPE来覆写一个特定的Scope，例如:

```java
import com.google.inject.Scopes;

@RunWith(Junit4.class)
final class FooTest {
  static class TestModule extends AbstractModule {
    @Override
    protected void configure() {
      bindScope(BatchScoped.class, Scopes.NO_SCOPE);
    }
  }
}
```

### 自定义Scope

一般建议用户不要编写自己的自定义作用域 — 内置作用域对于大多数应用程序来说应该足够了。

创建自定义作用域过程如下:

1. 定义Scope注解
2. 实现Scope接口
3. 将Scope注解附加到Scope接口实现类
4. 触发Scope，手动进入和退出

#### 定义Scope注解

```java
import static java.lang.annotation.ElementType.METHOD;
import static java.lang.annotation.ElementType.TYPE;
import java.lang.annotation.Retention;
import static java.lang.annotation.RetentionPolicy.RUNTIME;
import java.lang.annotation.Target;

@Target({ TYPE, METHOD })
@Retention(RUNTIME)
@ScopeAnnotation
public @interface BatchScoped {}
```

#### 实现Scope接口

```java
public interface Scope {

  /**
   * Scopes a provider. The returned provider returns objects from this scope.
   *  If an object does not exist in this scope, the provider can use the given unscoped provider to retrieve one.
   */
  public <T> Provider<T> scope(Key<T> key, Provider<T> unscoped);

  @Override
  String toString();
}
```

#### 注册 Scope

在模块的方法中将Scope注解附加到相应的Scope实现。

```java
public final class BatchScopeModule extends AbstractModule {
  // implements Scope
  private final SimpleScope batchScope = new SimpleScope();

  @Override
  protected void configure() {
    // tell Guice about the scope
    bindScope(BatchScoped.class, batchScope);
  }

  @Provides
  @Named("batchScope")
  SimpleScope provideBatchScope() {
    return batchScope;
  }
}

public final class ApplicationModule extends AbstractModule {
  @Override
  protected void configure() {
    install(new BatchScopeModule());
  }

  // Foo is bound in BatchScoped
  @Provides
  @BatchScoped
  Foo provideFoo() {
    return FooFactory.createFoo();
  }
}
```

#### 触发Scope

自定义Scope实现要求您手动进入和退出作用域。 通常这存在于一些低级基础设施代码中（比如Web服务器的入口点）。

```java
  @Inject @Named("batchScope") SimpleScope scope;
  public void scopeRunnable(Runnable runnable) {
    // 这里的enter和exit方法代指自定义Scope的触发方法，实际名称用户自己决定。
    scope.enter();
     try {
      // create and access scoped objects
      runnable.run();
    } finally {
      scope.exit();
    }
  }
```

## 绑定

绑定可以理解为Guice Map中的元素。通过创建绑定，我们可以像Guice Map中添加元素。下面我们来介绍几种类型的绑定。

### 链接绑定

链接绑定将一个类型绑定到其实现上，例如:

```java
public class BillingModule extends AbstractModule {
  @Provides
  TransactionLog provideTransactionLog(DatabaseTransactionLog impl) {
    return impl;
  }
}
```

然后我们可以使用`injector.getInstance(TransactionLog.class)`获取其实现类。链接绑定支持了链式定义:

```java
public class BillingModule extends AbstractModule {
  @Provides
  TransactionLog provideTransactionLog(DatabaseTransactionLog databaseTransactionLog) {
    return databaseTransactionLog;
  }

  @Provides
  DatabaseTransactionLog provideDatabaseTransactionLog(MySqlDatabaseTransactionLog impl) {
    return impl;
  }
}
```

通过上面的定义，我们可以将TransactionLog的实现类定义为 MySqlDatabaseTransactionLog。

除了使用`@Provides`注解声明链接绑定，还可以使用bind方法实现:

```java
bind(DatabaseTransactionLog.class).to(MySqlDatabaseTransactionLog.class);
```

### 注解绑定

当我们需要将多个同一类型的对象注入不同对象的时候，就需要使用注解区分这些依赖了。最简单的办法就是使用`@Named`注解进行区分。

首先需要在要注入的地方添加`@Named`注解。

```java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@Named("Checkout") CreditCardProcessor processor,
      TransactionLog transactionLog) {
    ...
  }
```

然后在绑定中添加annotatedWith方法指定`@Named`中指定的名称。由于编译器无法检查字符串，所以Guice官方建议我们保守地使用这种方式。

```java
bind(CreditCardProcessor.class)
  .annotatedWith(Names.named("Checkout"))
  .to(CheckoutCreditCardProcessor.class);
```

如果希望使用类型安全的方式，可以自定义注解。

```java
@Qualifier
@Target({ FIELD, PARAMETER, METHOD })
@Retention(RUNTIME)
public @interface PayPal {}
```

然后在需要注入的类上应用。

```java
public class RealBillingService implements BillingService {

  @Inject
  public RealBillingService(@PayPal CreditCardProcessor processor,
      TransactionLog transactionLog) {
    ...
  }
```

最后在配置类中，使用bind方法配置。

```java
bind(CreditCardProcessor.class)
  .annotatedWith(PayPal.class)
  .to(PayPalCreditCardProcessor.class);
```

### 实例绑定

有时候需要直接注入一个对象的实例，而不是从依赖关系中解析。如果我们要注入基本类型的话只能这么做。

```java
bind(String.class)
  .annotatedWith(Names.named("JDBC URL"))
  .toInstance("jdbc:mysql://localhost/pizza");
bind(Integer.class)
  .annotatedWith(Names.named("login timeout seconds"))
  .toInstance(10);
```

如果使用toInstance方法注入的实例比较复杂的话，可能会影响程序启动。这时候可以使用`@Provides`方法代替。

也可以使用`bindConstant`方法来绑定实例。`bindConstant`方法用于绑定基本类型和其他常量类型例如String,enum和Class

```java
bindConstant()
  .annotatedWith(HttpPort.class)
  .to(8080);
```

### `@Provides`方法

当一个对象很复杂，无法使用简单的构造器来生成的时候，我们可以使用`@Provides`方法，也就是在配置类中生成一个注解了`@Provides`的方法(可以是静态方法或者是实例方法)。在该方法中我们可以编写任意代码来构造对象。

```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    ...
  }

  @Provides
  TransactionLog provideTransactionLog() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setJdbcUrl("jdbc:mysql://localhost/pizza");
    transactionLog.setThreadPoolSize(30);
    return transactionLog;
  }
}
```

`@Provides`方法也可以应用`@Named`和自定义注解，还可以注入其他依赖，Guice会在调用方法之前注入需要的对象。

```java
@Provides @PayPal
CreditCardProcessor providePayPalCreditCardProcessor(
    @Named("PayPal API key") String apiKey) {
  PayPalCreditCardProcessor processor = new PayPalCreditCardProcessor();
  processor.setApiKey(apiKey);
  return processor;
}
```

默认Guice不允许异常从provider(或@Providers方法)中抛出。如果需要抛出异常，可以使用 Guice的 ThrowingProviders extension。

### Provider绑定

如果项目中存在多个比较复杂的对象需要构建，使用`@Provide`方法会让配置类变得比较乱。我们可以使用Guice提供的Provider接口将复杂的代码放到单独的类中。办法很简单，实现`Provider<T>`接口的get方法即可。在Provider类中，我们可以使用`@Inject`任意注入对象。

```java
public class DatabaseTransactionLogProvider implements Provider<TransactionLog> {
  private final Connection connection;

  @Inject
  public DatabaseTransactionLogProvider(Connection connection) {
    this.connection = connection;
  }

  public TransactionLog get() {
    DatabaseTransactionLog transactionLog = new DatabaseTransactionLog();
    transactionLog.setConnection(connection);
    return transactionLog;
  }
}
```

在配置类中使用toProvider方法绑定到Provider上即可。

```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class)
        .toProvider(DatabaseTransactionLogProvider.class);
  }
```
#### 注入Provider

在使用正常的依赖注入时，每个类型只会获得注入后的同一个实例，当想获取多个依赖类型的实例时，可以使用注入Provider的方法。例如，下面的LogFileEntry就有多个实例。

```java
public class LogFileTransactionLog implements TransactionLog {

  private final Provider<LogFileEntry> logFileProvider;

  @Inject
  public LogFileTransactionLog(Provider<LogFileEntry> logFileProvider) {
    this.logFileProvider = logFileProvider;
  }

  public void logChargeResult(ChargeResult result) {
    LogFileEntry summaryEntry = logFileProvider.get();
    summaryEntry.setText("Charge " + (result.wasSuccessful() ? "success" : "failure"));
    summaryEntry.save();

    if (!result.wasSuccessful()) {
      LogFileEntry detailEntry = logFileProvider.get();
      detailEntry.setText("Failure result: " + result);
      detailEntry.save();
    }
  }
```

当依赖实例非常昂贵时，注入Provider还能够实现懒加载。

```java
public class DatabaseTransactionLog implements TransactionLog {

  private final Provider<Connection> connectionProvider;

  @Inject
  public DatabaseTransactionLog(Provider<Connection> connectionProvider) {
    this.connectionProvider = connectionProvider;
  }

  public void logChargeResult(ChargeResult result) {
    /* only write failed charges to the database */
    if (!result.wasSuccessful()) {
      Connection connection = connectionProvider.get();
    }
  }
```

如上面Scope部分所说，注入provider也可以用于Scope间适配。

```java
@Singleton
public class ConsoleTransactionLog implements TransactionLog {

  private final AtomicInteger failureCount = new AtomicInteger();
  private final Provider<User> userProvider;

  @Inject
  public ConsoleTransactionLog(Provider<User> userProvider) {
    this.userProvider = userProvider;
  }

  public void logConnectException(UnreachableException e) {
    failureCount.incrementAndGet();
    User user = userProvider.get();
    System.out.println("Connection failed for " + user + ": " + e.getMessage());
    System.out.println("Failure count: " + failureCount.incrementAndGet());
  }
```

在这种情况下，User不会只注入一次，会在每次请求来时，提供不同的User。



### Untargeted 绑定

Untargeted Binding用于注册绑定没有目标（实现）类型的特化场景，一般是没有实现接口的普通类型，在没有使用@Named注解或者自定义注解绑定的前提下可以忽略to()调用。但是如果使用了@Named注解或者自定义注解进行绑定，to()调用一定不能忽略。例如：

```java
  bind(Foo.class).in(Scopes.SINGLETON);
  bind(Bar.class)
    .annotatedWith(Names.named("bar"))
    .to(Bar.class)
    .in(Scopes.SINGLETON);
```

我们可以先通过 Untargeted 绑定，将绑定类型的信息通知 injector，然后在使用注解绑定的时候，我们再将其绑定到具体的实现类上：

```java
bind(Foo.class).in(Scopes.SINGLETON);
bind(Foo.class)
    .annotatedWith(Names.named("foo"))
    .to(Foo.class)
    .in(Scopes.SINGLETON);
```

### 构造器绑定

有些时候我们需要把一种类型绑定到任意的构造器上。这个通常出现在@Inject注解无法应用于目标构造器的情况：要么因为它是一个第三方类，要么是因为它有多个构造器参与了依赖注入。@Provides方法提供了最佳的解决方案。通过显式调用我们自己的对象构造方法，我们就可以得到相应的类型了。但是这种方法有个缺陷：手动的构造的对象无法参与到AOP中。

为了解决这个问题，Guice提供了toConstructor()绑定的方式。这种方式需要我们通过反射选择一个类的构造器。而且，当构造器不存在的时候我们需要自己解决异常：

```java
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    try {
      bind(TransactionLog.class).toConstructor(
          DatabaseTransactionLog.class.getConstructor(DatabaseConnection.class));
    } catch (NoSuchMethodException e) {
      addError(e);
    }
  }
}
```

在这个例子中，DatabaseTransactionLog 类应该得有一个只接受一个DatabaseConnection类型参数的构造器。这个构造器不需要@Inject注解。Guice会调用它的构造器来适配对应的绑定。

每个toConstructor()绑定都是独立的。如果我们将一种单例类型绑定在多个构造器上，那么每个构造器都会创建出它们对应的对象。

### 内置绑定

除了显式绑定和即时绑定之外，Guice也内嵌了一些其他的绑定。

#### loggers

Guice专门为java.util.logging.Logger内嵌了一个绑定，这样可以节省一些我们的开发时间。这个绑定会自动把logger的名字设置成logger被注入的类名。

```java
@Singleton
public class ConsoleTransactionLog implements TransactionLog {

  private final Logger logger;

  @Inject
  public ConsoleTransactionLog(Logger logger) {
    this.logger = logger;
  }

  public void logConnectException(UnreachableException e) {
    /* the message is logged to the "ConsoleTransacitonLog" logger */
    logger.warning("Connect exception failed, " + e.getMessage());
  }
```

#### Injector

在我们的框架代码中，有的时候直到代码运行时我们才知道哪一种类型是我们需要的。在这种特殊情况下，我们就需要注入injector了。需要注意的是，注入injector的代码不会自行记录它的依赖关系，所以我们需要谨慎地使用这种方式。

##### Providers

Guice可以为所有感知到类型注入一个Provider。在Injectiong Providers中可以了解实现的细节。

#### TypeLiterals

Guice对其注入的所有类型都具有完整的类型信息。如果我们需要参数化这些类型，我们可以注入一个TypeLiteral<T>。这样我们就可以通过一些反射机制获取这些元素的类型了。

#### The Stage

Guice支持不同的阶段枚举类型，这样就可以区分开发环境和生产环境了。

#### MembersInjectors

当我们绑定providers或者扩展一些extensions的时候，我们可能需要Guice为我们写的对象注入依赖。想要做到这一点，只要添加一个`MemberInjector<T>`(其中T是我们自己的对象类型),然后通过调用membersInjector.injectMembers(myNewObject)就可以了。

### JIT 绑定

当injector需要一种类型的对象实例时，它就需要指定绑定。在modules中的绑定，我们称之为显式绑定，而我们想用的时候就可以通过这些modules创建injector。当我们需要的一个类型没有通过显示绑定到实现类上时，injector就会尝试创建一个即使绑定(Just-In_Time buding),这也被称之为JIT绑定或隐式绑定。

> 调用Binder#requireExplicitBindings()方法可以声明Module内必须显式声明所有绑定，也就是禁用隐式绑定，所有绑定必须在Module的实现中声明。
#### @Inject 构造函数

隐式绑定需要满足：

- 构造函数必须无参，并且非private修饰
- 注入器尚未选择要求显式@Inject构造函数，没有在Module实现中激活Binder#requireAtInjectRequired()，调用Binder#requireAtInjectRequired()方法可以强制声明Guice只使用带有@Inject注解的构造器。

例如:

```java
public final class Foo {
  // An @Inject annotated constructor.
  @Inject
  Foo(Bar bar) {
    ...
  }
}

public final class Bar {
  // A no-arg non private constructor.
  Bar() {}

  private static class Baz {
    // A private constructor to a private class is also usable by Guice, but
    // this is not recommended since it can be slow.
    private Baz() {}
  }
}
```

在以下情况下，构造函数不可注入：

- 构造函数采用一个或多个参数，并且不使用@Inject进行批注。
- 有多个带@Inject注释的构造函数。
- 构造函数在非静态嵌套类中定义。内部类有对无法注入的封闭类的隐式引用。

```java
public final class Foo {
  // Not injectable because the construct takes an argument and there is no
  // @Inject annotation.
  Foo(Bar bar) {
    ...
  }
}

public final class Bar {
  // Not injectable because the constructor is private
  private Bar() {}

  class Baz {
    // Not injectable because Baz is not a static inner class
    Baz() {}
  }
}
```

#### @ImplementedBy

注释的作用类似于链接绑定,指定生成类型时要使用的子类型。

```java
@ImplementedBy(PayPalCreditCardProcessor.class)
public interface CreditCardProcessor {
  ChargeResult charge(String amount, CreditCard creditCard)
      throws UnreachableException;
}
```

上面的注释等效于以下语句：bind()

```java
bind(CreditCardProcessor.class).to(PayPalCreditCardProcessor.class);
```
#### @ProvidedBy

@ProvidedBy告诉注入器一个产生Provider实例：

```java
@ProvidedBy(DatabaseTransactionLogProvider.class)
public interface TransactionLog {
  void logConnectException(UnreachableException e);
  void logChargeResult(ChargeResult result);
}
```

注释等效于绑定：toProvider()

```java
bind(TransactionLog.class)
  .toProvider(DatabaseTransactionLogProvider.class);
```

### 多重绑定

Multi Binding也就是多（实例）绑定，使用特化的Binder代理完成，这三种Binder代理分别是：

- Multibinder：可以简单理解为`Type => Set<TypeImpl>`，注入类型为`Set<Type>`
- MapBinder：可以简单理解为`(KeyType, ValueType) => Map<KeyType, ValueTypeImpl>`，注入类型为`Map<KeyType, ValueType>`
- OptionalBinder：可以简单理解为`Type => Optional.ofNullable(GuiceMap.get(Type)).or(DefaultImpl)`，注入类型为`Optional<Type>`

Multibinder的使用例子：

```java
public class GuiceMultiBinderDemo {

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new AbstractModule() {
            @Override
            public void configure() {
                Multibinder<Processor> multiBinder = Multibinder.newSetBinder(binder(), Processor.class);
                multiBinder.permitDuplicates().addBinding().to(FirstProcessor.class).in(Scopes.SINGLETON);
                multiBinder.permitDuplicates().addBinding().to(SecondProcessor.class).in(Scopes.SINGLETON);
            }
        });
        injector.getInstance(Client.class).process();
    }

    @Singleton
    public static class Client {

        @Inject
        private Set<Processor> processors;

        public void process() {
            Optional.ofNullable(processors).ifPresent(ps -> ps.forEach(Processor::process));
        }
    }

    interface Processor {
        void process();
    }

    public static class FirstProcessor implements Processor {

        @Override
        public void process() {
            System.out.println("FirstProcessor process...");
        }
    }

    public static class SecondProcessor implements Processor {

        @Override
        public void process() {
            System.out.println("SecondProcessor process...");
        }
    }
}
// 输出结果
FirstProcessor process...
SecondProcessor process...
```

MapBinder的使用例子：

```java
public class GuiceMapBinderDemo {

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new AbstractModule() {
            @Override
            public void configure() {
                MapBinder<Type, Processor> mapBinder = MapBinder.newMapBinder(binder(), Type.class, Processor.class);
                mapBinder.addBinding(Type.SMS).to(SmsProcessor.class).in(Scopes.SINGLETON);
                mapBinder.addBinding(Type.MESSAGE_TEMPLATE).to(MessageTemplateProcessor.class).in(Scopes.SINGLETON);
            }
        });
        injector.getInstance(Client.class).process();
    }

    @Singleton
    public static class Client {

        @Inject
        private Map<Type, Processor> processors;

        public void process() {
            Optional.ofNullable(processors).ifPresent(ps -> ps.forEach(((type, processor) -> processor.process())));
        }
    }

    public enum Type {
        SMS,
        MESSAGE_TEMPLATE
    }

    interface Processor {
        void process();
    }

    public static class SmsProcessor implements Processor {

        @Override
        public void process() {
            System.out.println("SmsProcessor process...");
        }
    }

    public static class MessageTemplateProcessor implements Processor {

        @Override
        public void process() {
            System.out.println("MessageTemplateProcessor process...");
        }
    }
}
// 输出结果
SmsProcessor process...
MessageTemplateProcessor process...
```

OptionalBinder的使用例子：

```java
public class GuiceOptionalBinderDemo {

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new AbstractModule() {
            @Override
            public void configure() {
              // 不带可选值的
//                bind(Logger.class).to(LogbackLogger.class).in(Scopes.SINGLETON);
                // 带可选值的
                OptionalBinder.newOptionalBinder(binder(), Logger.class)
                        .setDefault()
                        .to(StdLogger.class)
                        .in(Scopes.SINGLETON);
            }
        });
        injector.getInstance(Client.class).log("Hello World");
    }

    @Singleton
    public static class Client {

        @Inject
        private Optional<Logger> logger;

        public void log(String content) {
            logger.ifPresent(l -> l.log(content));
        }
    }


    interface Logger {

        void log(String content);
    }

    public static class StdLogger implements Logger {

        @Override
        public void log(String content) {
            System.out.println(content);
        }
    }
}
```

使用注解进行多重绑定:

1. 提供Set

```java
public class FlickrPluginModule extends AbstractModule {
  @ProvidesIntoSet
  UriSummarizer provideFlickerUriSummarizer() {
    return new FlickrPhotoSummarizer(...);
  }
}
```
2. 提供map:

```java
public class FlickrPluginModule extends AbstractModule {
  @StringMapKey("Flickr")
  @ProvidesIntoMap
  UriSummarizer provideFlickrUriSummarizer() {
    return new FlickrPhotoSummarizer(...);
  }
}
```

也可以自定义MapKey注解:

```java
@MapKey(unwrapValue=true)
@Retention(RUNTIME)
public @interface MyCustomEnumKey {
  MyCustomEnum value();
}
```
如果为 unwrapValue = true，则自定义批注的值用作键，否则使用整个注解作为键。

3. 提供Option

目前注解不能用于创建不存在/空可选绑定。

```java
public class FrameworkModule extends AbstractModule {
  @ProvidesIntoOptional(ProvidesIntoOptional.Type.DEFAULT)
  // 默认值
  @Singleton
  RequestLogger provideConsoleLogger() {
    return new DefaultRequestLoggerImpl();
  }
}
public class RequestLoggingModule extends AbstractModule {
  @ProvidesIntoOptional(ProvidesIntoOptional.Type.ACTUAL)
  //ACTUAL 用于覆盖默认值
  @Singleton
  RequestLogger provideConsoleLogger() {
    return new ConsoleLogger(System.out);
  }
}
```

### 限制绑定源

如果您拥有提供一组绑定的 Guice 库，则可能需要 防止库外部的代码提供它们。

```java
// Modules annotated with this Permit can provide Network bindings.
@RestrictedBindingSource.Permit
@Retention(RetentionPolicy.RUNTIME)
@interface NetworkPermit {}

// Bindings with the @IpAddress qualifier annotation can only be provided by
// modules with the NetworkPermit annotation.
@RestrictedBindingSource(
  explanation = "Please install NetworkModule instead of binding network bindings yourself.",
  permits = {NetworkPermit.class})
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
public @interface IpAddress {}

// The RoutingTable binding can only be provided by modules annotated with the
// NetworkPermit annotation.
@RestrictedBindingSource(
  explanation = "Please install NetworkModule instead of binding network bindings yourself.",
  permits = {NetworkPermit.class})
public interface RoutingTable {
  int getNextHopIpAddress(int destinationIpAddress);
}

@NetworkPermit
public final class NetworkModule extends AbstractModule {
  @Provides @IpAddress int provideIp( ... ) { ... }

  @Override
  protected void configure() {
    // RoutingModule is permitted to provide the RoutingTable binding because
    // it is installed by NetworkModule, which is annotated with NetworkPermit
    // - ie. it's enough for any module providing a binding (directly or
    // indirectly) to have the right permit.
    install(new RoutingModule());
  }
}

private final RoutingModule extends AbstractModule {
  @Provides RoutingTable provideRoutingTable( ... ) { ... }
}
```

## AOP

Guice提供了相对底层的AOP特性，使用者需要自行实现org.aopalliance.intercept.MethodInterceptor接口在方法执行点的前后插入自定义代码，并且通过Binder#bindInterceptor()注册方法拦截器。这里只通过一个简单的例子进行演示，模拟的场景是方法执行前和方法执行完成后分别打印日志，并且计算目标方法调用耗时：

```java
public class GuiceAopDemo {

    public static void main(String[] args) {
        Injector injector = Guice.createInjector(new AbstractModule() {
            @Override
            public void configure() {
                bindInterceptor(Matchers.only(EchoService.class), Matchers.any(), new EchoMethodInterceptor());
            }
        });
        EchoService instance = injector.getInstance(Key.get(EchoService.class));
        instance.echo("throwable");
    }

    public static class EchoService {

        public void echo(String name) {
            System.out.println(name + " echo");
        }
    }

    public static class EchoMethodInterceptor implements MethodInterceptor {

        @Override
        public Object invoke(MethodInvocation methodInvocation) throws Throwable {
            Method method = methodInvocation.getMethod();
            String methodName = method.getName();
            long start = System.nanoTime();
            System.out.printf("Before invoke method => [%s]\n", methodName);
            Object result = methodInvocation.proceed();
            long end = System.nanoTime();
            System.out.printf("After invoke method => [%s], cost => %d ns\n", methodName, (end - start));
            return result;
        }
    }
}

// 输出结果
Before invoke method => [echo]
throwable echo
After invoke method => [echo], cost => 16013700 ns
```

Guice AOP 动态创建一个子类(字节码生成)，该子类通过重写方法应用拦截器。aop 存在一些局限性:

- 类必须是公共类或包私有类。
- 类必须是非final的。
- 方法必须是公共的、包私有的或受保护的
- 方法必须是非final的。
- 实例必须由Guice通过@Inject注释或无参数构造器创建。无法在不是由Guice建造的实例上使用方法拦截。

### 注入拦截器

如果需要将依赖项注入拦截器:

```java
public class AopModule extends AbstractModule {
  protected void configure() {
    EchoMethodInterceptor echo = new EchoMethodInterceptor();
    requestInjection(echo);
    bindInterceptor(Matchers.only(EchoService.class), Matchers.any(),
       echo);
  }
}
```

还可以使用 Binder.getProvider 并将依赖项传递到 拦截器的构造函数。

```java
public class AopModule extends AbstractModule {
  protected void configure() {
    bindInterceptor(Matchers.only(EchoService.class), Matchers.any(),
                    new EchoMethodInterceptor(getProvider(XXXX.class)));
  }
}
```

注入拦截器时要小心。防止出现递归问题。

## 定制注入

通过TypeListener和MembersInjector可以实现目标类型实例的成员属性自定义注入扩展。例如可以通过下面的方式实现目标实例的org.slf4j.Logger属性的自动注入

```java
public class GuiceCustomInjectionDemo {

    public static void main(String[] args) throws Exception {
        Injector injector = Guice.createInjector(new AbstractModule() {
            @Override
            public void configure() {
                bindListener(Matchers.any(), new LoggingListener());
            }
        });
        injector.getInstance(LoggingClient.class).doLogging("Hello World");
    }

    public static class LoggingClient {

        @Logging
        private Logger logger;

        public void doLogging(String content) {
            Optional.ofNullable(logger).ifPresent(l -> l.info(content));
        }
    }

    @Qualifier
    @Retention(RUNTIME)
    @interface Logging {

    }

    public static class LoggingMembersInjector<T> implements MembersInjector<T> {

        private final Field field;
        private final Logger logger;

        public LoggingMembersInjector(Field field) {
            this.field = field;
            this.logger = LoggerFactory.getLogger(field.getDeclaringClass());
            field.setAccessible(true);
        }

        @Override
        public void injectMembers(T instance) {
            try {
              //设置记录器
                field.set(instance, logger);
            } catch (IllegalAccessException e) {
                throw new IllegalStateException(e);
            } finally {
                field.setAccessible(false);
            }
        }
    }
    //实现 TypeListener 来扫描类型的字段，查找 Log4J 记录器。
    public static class LoggingListener implements TypeListener {

        @Override
        public <I> void hear(TypeLiteral<I> typeLiteral, TypeEncounter<I> typeEncounter) {
            Class<?> clazz = typeLiteral.getRawType();
            while (Objects.nonNull(clazz)) {
                for (Field field : clazz.getDeclaredFields()) {
                    if (field.getType() == Logger.class && field.isAnnotationPresent(Logging.class)) {
                        typeEncounter.register(new LoggingMembersInjector<>(field));
                    }
                }
                clazz = clazz.getSuperclass();
            }
        }
    }
}
// 输出结果
[2022-02-22 00:51:33,516] [INFO] cn.vlts.guice.GuiceCustomInjectionDemo$LoggingClient [main] [] - Hello World
```

## 参考

- [1] [Guice Wiki](https://github.com/google/guice/wiki/)
- [2] [aopalliance](https://aopalliance.sourceforge.net/)

