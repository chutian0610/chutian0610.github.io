# 设计模式之控制反转

控制反转（Inversion of Control）是一种是面向对象编程中的一种设计原则，用来减低计算机代码之间的耦合度。其基本思想是：借助于“第三方”实现具有依赖关系的对象之间的解耦。

<!--more-->

在传统编程中，业务逻辑的流由静态绑定的对象确定。通过反转控制，流取决于在程序执行期间构建的对象图。这种动态流是通过抽象定义的对象交互实现的。此运行时绑定是通过依赖项注入或服务定位器等机制实现的。在 IoC 中，代码也可以在编译期间静态链接，但通过从外部配置读取其描述而不是在代码本身中直接引用来查找要执行的代码。

我们来看下，为什么说控制反转(IOC)是一种解耦：

- 在没有引入IOC容器之前，对象A依赖于对象B，那么对象A在初始化或者运行到某一点的时候，自己必须主动去创建对象B或者使用已经创建的对象B。无论是创建还是使用对象B，控制权都在自己手上。
- 软件系统在引入IOC容器之后，这种情形就完全改变了，由于IOC容器的加入，对象A与对象B之间失去了直接联系，所以，当对象A运行到需要对象B的时候，IOC容器会主动创建一个对象B注入到对象A需要的地方。

通过前后的对比，我们不难看出来：对象A获得依赖对象B的过程,由主动行为变为了被动行为，控制权颠倒过来了，这就是“控制反转”这个名称的由来[^1]。


## 控制反转的实现

在看实现前，先看一个简单的场景:

我们有一个订单系统，需要一个操作日志记录服务，记录我们的订单操作日志。一个非常简单的方案如下:

```mermaid
classDiagram
    class BillService
    class TransactionLogger{
      * logChargeEvent(ChargeEvent chargeEvent)
    }
    &lt;&lt;interface>> TransactionLogger
    class DbTransactionLogger{
      + logChargeEvent(ChargeEvent chargeEvent)
    }
    DbTransactionLogger..|>	TransactionLogger: implement
    BillService..>TransactionLogger:depend
    BillService..>DbTransactionLogger:create
```

在(服务)BillService的实例创建时，会同时创建一个(组件)TransactionLogger的实例。将场景扩展到一个真正的系统，我们可能会有几十个这样的服务和组件。通过控制反转，容器可以控制如何将组件组装为服务。

在面向对象的编程中，有几种基本技术可以实现控制反转:

- 使用服务定位器模式
- 使用依赖注入
- 使用模板方法设计模式
- 使用策略设计模式

下面会介绍几种方法的具体实现。

### 依赖注入

依赖注入的基本思想是有一个单独的服务，一个注入器，用于填充服务类中的字段接口的实现。

```mermaid
classDiagram
    class BillService
    class Assembler
    class TransactionLogger{
      * logChargeEvent(ChargeEvent chargeEvent)
    }
    &lt;&lt;interface>> TransactionLogger
    class DbTransactionLogger{
      + logChargeEvent(ChargeEvent chargeEvent)
    }
    DbTransactionLogger..|>	TransactionLogger: implement
    BillService..>TransactionLogger
    Assembler..>DbTransactionLogger:create
    Assembler..>TransactionLogger:depend
    Assembler..>BillService:depend
```

依赖关系注入器的主要优点是它删除了服务对具体组件实现的依赖关系。

### 服务定位器

服务定位器需要具有返回对应组件实现的方法。

```mermaid
classDiagram
    class BillService
    class Assembler
    class ServiceLocator
    class TransactionLogger{
      * logChargeEvent(ChargeEvent chargeEvent)
    }
    &lt;&lt;interface>> TransactionLogger
    class DbTransactionLogger{
      + logChargeEvent(ChargeEvent chargeEvent)
    }
    DbTransactionLogger..|>	TransactionLogger: implement
    BillService..>TransactionLogger
    BillService..>ServiceLocator:depend
    Assembler..>DbTransactionLogger:create
    Assembler..>TransactionLogger:depend
    Assembler..>BillService:depend
    Assembler..>ServiceLocator:depend
```

一个典型的服务定位器的例子是java中的JNI。JDK中的ServiceLoader是使用配置文件的服务定位器的例子。

### 模板方法

模板方法在前面的设计模式中有单独介绍。此处就不做赘述。

模板方法用于框架中，其中每个框架实现领域体系结构的固定部分，同时提供用于自定义的钩子方法。这是控制反转的一个例子。

- 它允许子类实现不同的行为（通过重写钩子方法）。
- 它避免了代码中的重复：算法的一般工作流在抽象类的模板方法中实现一次，并且在子类中实现必要的变体。
- 它允许专业化的点控制。如果子类只是重写模板方法，则它们可能会对工作流进行根本和任意的更改。相比之下，通过仅覆盖钩子方法，只能更改工作流的某些特定细节，并且整个工作流保持不变。

### 策略模式

策略模式在前面的设计模式中有单独介绍。此处就不做赘述。策略模式可以在运行时选择算法，这也是控制反转的提现。


[^1]:控制反转和依赖注入的区别: IoC框架使用依赖注入作为实现控制反转的方式，但是控制反转还有其他的实现方式，所以不能将控制反转和依赖注入等同。

