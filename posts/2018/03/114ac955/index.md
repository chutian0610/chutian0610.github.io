# Java并发之Lock简介

通过查看Lock的源码可知，Lock是一个接口：

```java
public interface Lock {
    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```

lock()、tryLock()、tryLock(long time, TimeUnit unit)和lockInterruptibly()是用来获取锁的。unLock()方法是用来释放锁的。

<!--more-->

lock()方法是平常使用得最多的一个方法，就是用来获取锁。如果锁已被其他线程获取，则进行等待。如果采用Lock，必须主动去释放锁，并且在发生异常时，不会自动释放锁。因此一般来说，使用Lock必须在try{}catch{}块中进行，并且将释放锁的操作放在finally块中进行，以保证锁一定被被释放，防止死锁的发生。通常使用Lock来进行同步的话，是以下面这种形式去使用的：

```java
Lock lock = ...;
lock.lock();
try{
    //处理任务
}catch(Exception ex){
    // 处理异常
}finally{
    lock.unlock();   //释放锁
}
```

tryLock()方法是有返回值的，它表示用来尝试获取锁，如果获取成功，则返回true，如果获取失败（即锁已被其他线程获取），则返回false，***也就说这个方法无论如何都会立即返回。在拿不到锁时不会一直在那等待。***

tryLock(long time, TimeUnit unit)方法和tryLock()方法是类似的，只不过区别在于这个方法在拿不到锁时会等待一定的时间，在时间期限之内如果还拿不到锁，就返回false。如果如果一开始拿到锁或者在等待期间内拿到了锁，则返回true。

```java
Lock lock = ...;
if(lock.tryLock()) {
    try{
        //处理任务
    }catch(Exception ex){
        // 处理异常
    }finally{
        lock.unlock();   //释放锁
    }
}else {
    //如果不能获取锁，则直接做其他事情
}
```

lockInterruptibly()方法比较特殊，当通过这个方法去获取锁时，如果线程正在等待获取锁，则这个线程能够响应中断，即中断线程的等待状态。也就使说，当两个线程同时通过lock.lockInterruptibly()想获取某个锁时，假若此时线程A获取到了锁，而线程B只有在等待，那么对线程B调用threadB.interrupt()方法能够中断线程B的等待过程。

由于lockInterruptibly()的声明中抛出了异常，所以lock.lockInterruptibly()必须放在try块中或者在调用lockInterruptibly()的方法外声明抛出InterruptedException。

```java
public void method() throws InterruptedException {
    lock.lockInterruptibly();
    try {
     //.....
    }
    finally {
        lock.unlock();
    }
}
```

>当一个线程获取了锁之后，是不会被interrupt()方法中断的。因为本身在前面的文章中讲过单独调用interrupt()方法不能中断正在运行过程中的线程，只能中断阻塞过程中的线程。

## ReentrantLock

ReentrantLock，意思是“可重入锁”，ReentrantLock是唯一实现了Lock接口的类。下面是一个简单的使用例子:

```java
class X {
   private final ReentrantLock lock = new ReentrantLock(); // 每个对象持有同一个lock变量
   public void m() {
     lock.lock();  // block until condition holds
     try {
       // ... method body
      finally {
       lock.unlock()
     }
   }
 }}
```

### 可重入锁

如果锁具备可重入性，则称作为可重入锁。像synchronized和ReentrantLock都是可重入锁。举个例子：当一个线程执行到某个synchronized方法时，比如说method1，而在method1中会调用另外一个synchronized方法method2，此时线程不必重新去申请锁，而是可以直接执行方法method2。

### 可中断锁

可中断锁：顾名思义，就是可以相应中断的锁。在Java中，synchronized就不是可中断锁，而Lock是可中断锁。

如果某一线程A正在执行锁中的代码，另一线程B正在等待获取该锁，可能由于等待时间过长，线程B不想等待了，想先处理其他事情，我们可以让它中断自己或者在别的线程中中断它，这种就是可中断锁。

### 公平锁

公平锁即尽量以请求锁的顺序来获取锁。比如同是有多个线程在等待一个锁，当这个锁被释放时，等待时间最久的线程（最先请求的线程）会获得该所，这种就是公平锁。非公平锁即无法保证锁的获取是按照请求锁的顺序进行的。这样就可能导致某个或者一些线程永远获取不到锁。在Java中，synchronized就是非公平锁，它无法保证等待的线程获取锁的顺序。而对于ReentrantLock和ReentrantReadWriteLock，它默认情况下是非公平锁，但是可以设置为公平锁。

我们可以在创建ReentrantLock对象时，通过以下方式来设置锁的公平性：

```java
ReentrantLock lock = new ReentrantLock(true);
// 如果参数为true表示为公平锁，为fasle为非公平锁。默认情况下，如果使用无参构造器，则是非公平锁。
```

## ReadWriteLock

ReadWriteLock也是一个接口，在它里面只定义了两个方法：

```java
public interface ReadWriteLock {
    /**
     * Returns the lock used for reading.
     *
     * @return the lock used for reading.
     */
    Lock readLock();
 
    /**
     * Returns the lock used for writing.
     *
     * @return the lock used for writing.
     */
    Lock writeLock();
}
```

一个用来获取读锁，一个用来获取写锁。也就是说将文件的读写操作分开，分成2个锁来分配给线程，从而使得多个线程可以同时进行读操作。ReentrantReadWriteLock实现了ReadWriteLock接口。

> 如果有一个线程已经占用了读锁，则此时其他线程如果要申请写锁，则申请写锁的线程会一直等待释放读锁。如果有一个线程已经占用了写锁，则此时其他线程如果申请写锁或者读锁，则申请的线程会一直等待释放写锁。

### 读写锁

读写锁将对一个资源（比如文件）的访问分成了2个锁，一个读锁和一个写锁。正因为有了读写锁，才使得多个线程之间的读操作不会发生冲突。ReadWriteLock就是读写锁，它是一个接口，ReentrantReadWriteLock实现了这个接口。可以通过readLock()获取读锁，通过writeLock()获取写锁。

## Condition

在Condition中，用await()替换wait()，用signal()替换notify()，用signalAll()替换notifyAll()，传统线程的通信方式，Condition都可以实现，这里注意，Condition是被绑定到Lock上的，要创建一个Lock的Condition必须用newCondition()方法。

Condition的强大之处在于它可以为多个线程间建立不同的Condition， 使用synchronized/wait()只有一个阻塞队列，notifyAll会唤起所有阻塞队列下的线程，而使用lock/condition，可以实现多个阻塞队列，signalAll只会唤起某个阻塞队列下的阻塞线程。

下面用两种方式编写生产者/消费者模式代码加以说明:

```java
class Buffer {
    private int maxSize;
    private List<Date> storage;
    Buffer(int size){
        maxSize=size;
        storage=new LinkedList<>();
    }
    //生产方法
    public synchronized void put()  {
        try {
            while (storage.size() ==maxSize ){//如果队列满了
                System.out.print(Thread.currentThread().getName()+": wait \n");;
                wait();//阻塞线程
            }
            storage.add(new Date());
            System.out.print(Thread.currentThread().getName()+": put:"+storage.size()+ "\n");
            Thread.sleep(1000);
            notifyAll();//唤起线程
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    //消费方法
    public synchronized void take() {
        try { 
            while (storage.size() ==0 ){//如果队列空了
                System.out.print(Thread.currentThread().getName()+": wait \n");;
                wait();//阻塞线程
            }
            Date d=((LinkedList<Date>)storage).poll();
            System.out.print(Thread.currentThread().getName()+": take:"+storage.size()+ "\n");
            Thread.sleep(1000);
            notifyAll();//唤起线程
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    } 
}

// **************************************************************************************
class Buffer {
    private  final Lock lock;
    private  final Condition notFull;
    private  final Condition notEmpty;
    private int maxSize;
    private List<Date> storage;
    Buffer(int size){
        //使用锁lock，并且创建两个condition，相当于两个阻塞队列
        lock=new ReentrantLock();
        notFull=lock.newCondition();
        notEmpty=lock.newCondition();
        maxSize=size;
        storage=new LinkedList<>();
    }
    public void put()  {
        lock.lock();
        try {
            while (storage.size() ==maxSize ){//如果队列满了
                System.out.print(Thread.currentThread().getName()+": wait \n");;
                notFull.await();//阻塞生产线程
            }
            storage.add(new Date());
            System.out.print(Thread.currentThread().getName()+": put:"+storage.size()+ "\n");
            Thread.sleep(1000);
            notEmpty.signalAll();//唤醒消费线程
        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally{
            lock.unlock();
        }
    }

    public  void take() {
        lock.lock();
        try {
            while (storage.size() ==0 ){//如果队列空了
                System.out.print(Thread.currentThread().getName()+": wait \n");;
                notEmpty.await();//阻塞消费线程
            }
            Date d=((LinkedList<Date>)storage).poll();
            System.out.print(Thread.currentThread().getName()+": take:"+storage.size()+ "\n");
            Thread.sleep(1000);
            notFull.signalAll();//唤醒生产线程

        } catch (InterruptedException e) {
            e.printStackTrace();
        }finally{
            lock.unlock();
        }
    }
}
```

