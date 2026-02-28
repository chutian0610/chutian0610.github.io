# 设计模式之依赖注入

依赖关系注入是一种软件设计模式，其中一个或多个依赖关系（或服务）被注入或通过引用传递到依赖对象（或客户端）中，并成为客户端状态的一部分。该模式将客户端依赖项的创建与其自身行为分开，这允许程序设计松散耦合并遵循控制反转和单一责任原则。

简单来说，依赖注入通过请求获取它们的子组件而不是通过创建它们来获取, 将依赖关系的创建与其自身行为分开。

<!--more-->

## 为什么需要依赖注入

假设我们有一个下单功能, 使用信用卡支付订单:

```java
public interface BillingService {
  Receipt chargeOrder(Order order, Card card);
}
```

下面是其实现(使用信用卡和记录事务日志的下单逻辑):

```java
public class RealBillingService implements BillingService {
  public Receipt chargeOrder(Order order, Card card){
    CreditCardProcessor processor = new CreditCardProcessor();
    TransactionLog transactionLog = new DatabaseTransactionLog();

    try {
      ChargeResult result = processor.charge(card, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

很快我们有了新的支付方式，会员卡支付。同时我们想把日志直接打印在文件中。很显然，上面的代码过于耦合，我们不应该在BillingService的构造器中初始化其子组件。

通过依赖注入的思想，我们可以将客户端和服务实现类分离来解决这个问题。在依赖注入中，我们引入了注入器(有时也称为容器、提供者或工厂)向客户端提供服务[^1]。

下面是一个通过构造器来实现依赖注入的例子:

```java
public class RealBillingService implements BillingService {
  private final CardProcessor processor;
  private final TransactionLog transactionLog;

  public RealBillingService(CardProcessor processor,
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(Order order, Card card) {
    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
```

TransactionLog 和 CreditCardProcessor 作为构造函数参数传入RealBillingService。我们可以随意传入不同的服务实现，来执行下单逻辑

## 依赖注入的类型

客户端可以通过三种主要方式接收注入的服务: 

- 构造函数注入，其中依赖项通过客户端的类构造函数提供。

```java
public class Client {
    private Service service;

    // The dependency is injected through a constructor.
    Client(Service service) {
        if (service == null) {
            throw new InvalidParameterException("service must not be null");
        }
        this.service = service;
    }
}
```

- setter注入(字段注入)，其中客户端提供接受依赖项的setter方法, 在某些语言中(例如Java)反射可以直接进行属性字段注入。

```java
public class Client {
    private Service service;

    // The dependency is injected through a setter method.
    public void setService(Service service) {
        if (service == null) {
            throw new InvalidParameterException("service must not be null");
        }
        this.service = service;
    }
}
```

- 接口注入, 也就是说将注入的代码放在了接口方法里，接口注入模式因为具备侵入性，它要求组件必须与特定的接口相关联，因此并不被看好，实际使用有限。

```java
// 注入接口
public interface ServiceSetter {
    public void setService(Service service);
}

public class Client implements ServiceSetter {
    private Service service;

    @Override
    public void setService(Service service) {
        if (service == null) {
            throw new InvalidParameterException("service must not be null");
        }
        this.service = service;
    }
}

public class ServiceInjector {
	private Set<ServiceSetter> clients;

	public void inject(ServiceSetter client) {
		this.clients.add(client);
		client.setService(new ExampleService());
	}

	public void switch() {
		for (Client client : this.clients) {
			client.setService(new AnotherExampleService());
		}
	}
}

public class ExampleService implements Service {}

public class AnotherExampleService implements Service {}
```

### 框架注入

对于大型项目，手动依赖注入通常很乏味且容易出错，从而促进了自动化流程的框架使用。一旦构造代码不再是应用程序自定义的，而是通用的，手动依赖关系注入就成为依赖关系注入框架。一些框架，如 Guice，可以使用配置来规划程序组合。

```java
// 在Guice模块配置服务和实现的映射
public class BillingModule extends AbstractModule {
  @Override
  protected void configure() {
    bind(TransactionLog.class).to(DatabaseTransactionLog.class);
    bind(CardProcessor.class).to(PaypalCreditCardProcessor.class);
    bind(BillingService.class).to(RealBillingService.class);
  }
}
// Guice 将检查带 @Inject 注释的构造函数，并查找每个构造函数的参数实现。
public class RealBillingService implements BillingService {
  private final CreditCardProcessor processor;
  private final TransactionLog transactionLog;

  @Inject
  public RealBillingService(CreditCardProcessor processor,
      TransactionLog transactionLog) {
    this.processor = processor;
    this.transactionLog = transactionLog;
  }

  public Receipt chargeOrder(PizzaOrder order, CreditCard creditCard) {
    try {
      ChargeResult result = processor.charge(creditCard, order.getAmount());
      transactionLog.logChargeResult(result);

      return result.wasSuccessful()
          ? Receipt.forSuccessfulCharge(order.getAmount())
          : Receipt.forDeclinedCharge(result.getDeclineMessage());
     } catch (UnreachableException e) {
      transactionLog.logConnectException(e);
      return Receipt.forSystemFailure(e.getMessage());
    }
  }
}
// 最后，Guice会帮我们进行依赖注入。
public static void main(String[] args) {
  Injector injector = Guice.createInjector(new BillingModule());
  BillingService billingService = injector.getInstance(BillingService.class);
}
```

## 优缺点

### 优点

- 依赖关系注入的一个基本好处是减少了类与其依赖关系之间的耦合。
- 通过消除客户端对其依赖项如何实现的知识，程序变得更加可重用、可测试和维护。
- 这也提高了灵活性：客户端可以对支持客户端期望的内部接口的任何内容进行操作。
- 依赖关系注入减少了样板代码，因为所有依赖关系的创建都由单个组件处理。
- 依赖注入允许并发开发。两个开发人员可以独立开发相互使用的类，而只需要知道类将通过哪些接口进行通信。

### 缺点

- 创建需要配置详细信息的客户端，当有明显的默认值可用时，这些详细信息可能很繁琐。
- 使代码难以跟踪，因为它将行为与构造分开。
- 通常通过反射或动态编程实现，阻碍了IDE自动化。
- 通常需要更多的前期开发工作。
- 鼓励对框架的依赖。


[^1]: 服务和客户端:服务是包含有用功能的任何类。反过来，客户端是使用服务的任何类。

