# 设计模式之命令模式


在软件开发中，我们经常需要向某些对象发送请求（调用其中的某个或某些方法），但是并不知道请求的接收者是谁，也不知道被请求的操作是哪个，此时，我们特别希望能够以一种松耦合的方式来设计软件，使得请求发送者与请求接收者能够消除彼此之间的耦合，让对象之间的调用关系更加灵活，可以灵活地指定请求接收者以及被请求的操作。

命令模式为此类问题提供了一个较为完美的解决方案。命令模式可以将请求发送者和接收者完全解耦，发送者与接收者之间没有直接引用关系，发送请求的对象只需要知道如何发送请求，而不必知道如何完成请求。

<!--more-->

## 命令模式定义

命令模式\(Command Pattern\)：将一个请求封装为一个对象，从而让我们可用不同的请求对客户进行参数化；对请求排队或者记录请求日志，以及支持可撤销的操作。命令模式是一种对象行为型模式，其别名为动作\(Action\)模式或事务\(Transaction\)模式。

```mermaid
classDiagram
class Invoker
Invoker: -Command command
Invoker *-- Command
class Command
&lt;&lt;abstract>> Command
Command: +execute()*
class ConcreteCommand
ConcreteCommand: -Receiver receiver
ConcreteCommand: +execute()
Command<|-- ConcreteCommand
class Receiver
Receiver: +action()
ConcreteCommand *-- Receiver
```

在命令模式结构图中包含如下几个角色：

* Command（抽象命令类）：抽象命令类一般是一个抽象类或接口，在其中声明了用于执行请求的execute()等方法，通过这些方法可以调用请求接收者的相关操作。
* ConcreteCommand（具体命令类）：具体命令类是抽象命令类的子类，实现了在抽象命令类中声明的方法，它对应具体的接收者对象，将接收者对象的动作绑定其中。在实现execute()方法时，将调用接收者对象的相关操作(Action)。
* Invoker（调用者）：调用者即请求发送者，它通过命令对象来执行请求。一个调用者并不需要在设计时确定其接收者，因此它只与抽象命令类之间存在关联关系。在程序运行时可以将一个具体命令对象注入其中，再调用具体命令对象的execute()方法，从而实现间接调用请求接收者的相关操作。
* Receiver（接收者）：接收者执行与请求相关的操作，它具体实现对请求的业务处理。

命令模式的本质是对请求进行封装，一个请求对应于一个命令，将发出命令的责任和执行命令的责任分割开。每一个命令都是一个操作：请求的一方发出请求要求执行一个操作；接收的一方收到请求，并执行相应的操作。命令模式允许请求的一方和接收的一方独立开来，使得请求的一方不必知道接收请求的一方的接口，更不必知道请求如何被接收、操作是否被执行、何时被执行，以及是怎么被执行的。

命令模式的关键在于引入了抽象命令类，请求发送者针对抽象命令类编程，只有实现了抽象命令类的具体命令才与请求接收者相关联。在最简单的抽象命令类中只包含了一个抽象的execute()方法，每个具体命令类将一个Receiver类型的对象作为一个实例变量进行存储，从而具体指定一个请求的接收者，不同的具体命令类提供了execute()方法的不同实现，并调用不同接收者的请求处理方法。

## 代码示例

```java
// 抽象命令类
public abstract class Command {  
    public abstract void execute();  
}

// 具体命令类
public class ConcreteCommandA extends Command{
  private Receiver receiver;  // 如果有多个接受者，此处为 List<Receiver> 

  public ConcreteCommandA(Receiver receiver){
    this.receiver = receiver;
  }

  public void execute(){
      this.receiver.action();
  }
}

// 抽象接受者
public abstract class Receiver{
  public abstract void action();
}

public class ConcreteReceiver{
  public void action(){
    // 具体业务
  }
}

// 调用者
public class Invocator{
  private Command command;
  //设值注入  
  public void setCommand(Command command){
    this.command = command;
  }
  //业务方法，用于调用命令类的execute()方法  
    public void call() {  
        command.execute();  
    }  
}
```

## 扩展

### 命令队列

有时候我们需要将多个请求排队，当一个请求发送者发送一个请求时，将不止一个请求接收者产生响应，这些请求接收者将逐个执行业务方法，完成对请求的处理。此时，我们可以通过命令队列来实现。

```java
public class CommandQueue {  
    //定义一个ArrayList来存储命令队列  
    private ArrayList<Command> commands = new ArrayList<Command>();  
      
    public void addCommand(Command command) {  
        commands.add(command);  
    }  
      
    public void removeCommand(Command command) {  
        commands.remove(command);  
    }  
      
    //循环调用每一个命令对象的execute()方法  
    public void execute() {  
        for (Object command : commands) {  
            ((Command)command).execute();  
        }  
    }  
}  
```

当一个发送者发送请求后，将有一系列接收者对请求作出响应，命令队列可以用于设计批处理应用程序，如果请求接收者的接收次序没有严格的先后次序，我们还可以使用多线程技术来并发调用命令对象的execute()方法，从而提高程序的执行效率。

### 宏命令

命令队列的进一步扩展是宏命令，宏命令是一个具体命令类，它拥有一个集合属性，在该集合中包含了对其他命令对象的引用。通常宏命令不直接与请求接收者交互，而是通过它的成员来调用接收者的方法。当调用宏命令的execute()方法时，将递归调用它所包含的每个成员命令的execute()方法，一个宏命令的成员可以是简单命令，还可以继续是宏命令。执行一个宏命令将触发多个具体命令的执行，从而实现对命令的批处理。

### 撤销操作的实现

在命令模式中，我们可以通过调用一个命令对象的execute()方法来实现对请求的处理，如果需要撤销(Undo)请求，可通过在命令类中增加一个逆向操作来实现。简单地单次撤销，只要维护execute前的目标状态，如果要实现多次撤销操作，可以通过引入一个命令集合或其他方式来存储每一次操作时命令的状态。除了Undo操作外，还可以采用类似的方式实现恢复(Redo)操作，即恢复所撤销的操作（或称为二次撤销）。

### 日志文件

undo，redo 的进一步扩展是日志文件， 可以将命令队列中的所有命令对象都存储在一个日志文件中，每执行一个命令则从日志文件中删除一个对应的命令对象，防止因为断电或者系统重启等原因造成请求丢失，而且可以避免重新发送全部请求时造成某些命令的重复执行，只需读取请求日志文件，再继续执行文件中剩余的命令即可。

## 总结

### 主要优点

1. 降低系统的耦合度。由于请求者与接收者之间不存在直接引用，因此请求者与接收者之间实现完全解耦，相同的请求者可以对应不同的接收者，同样，相同的接收者也可以供不同的请求者使用，两者之间具有良好的独立性。
2. 新的命令可以很容易地加入到系统中。由于增加新的具体命令类不会影响到其他类，因此增加新的具体命令类很容易，无须修改原有系统源代码，甚至客户类代码，满足“开闭原则”的要求。
3. 可以比较容易地设计一个命令队列或宏命令（组合命令）。
4. 为请求的撤销(Undo)和恢复(Redo)操作提供了一种设计和实现方案。
 
### 主要缺点

* 使用命令模式可能会导致某些系统有过多的具体命令类。因为针对每一个对请求接收者的调用操作都需要设计一个具体命令类，因此在某些系统中可能需要提供大量的具体命令类，这将影响命令模式的使用。
 
### 适用场景

在以下情况下可以考虑使用命令模式：

1. 系统需要将请求调用者和请求接收者解耦，使得调用者和接收者不直接交互。请求调用者无须知道接收者的存在，也无须知道接收者是谁，接收者也无须关心何时被调用。
2. 系统需要在不同的时间指定请求、将请求排队和执行请求。一个命令对象和请求的初始调用者可以有不同的生命期，换言之，最初的请求发出者可能已经不在了，而命令对象本身仍然是活动的，可以通过该命令对象去调用请求接收者，而无须关心请求调用者的存在性，可以通过请求日志文件等机制来具体实现。
3. 系统需要支持命令的撤销(Undo)操作和恢复(Redo)操作。
4. 系统需要将一组操作组合在一起形成宏命令。

