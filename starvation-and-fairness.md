# 饥饿和公平

如果一个线程因为 CPU 时间全部被其他线程抢走而得不到 CPU 运行时间，这种状态被称之为“饥饿”。而该线程被“饥饿致死”正是因为它得不到 CPU 运行时间的机会。解决饥饿的方案被称之为“公平性” – 即所有线程均能公平地获得运行机会。

## 下面是本文讨论的主题：

1. Java 中导致饥饿的原因：

 - 高优先级线程吞噬所有的低优先级线程的 CPU 时间。
 - 线程被永久堵塞在一个等待进入同步块的状态。
 - 线程在等待一个本身也处于永久等待完成的对象(比如调用这个对象的 wait 方法)。  
2. 在 Java 中实现公平性方案，需要: 

 - 使用锁，而不是同步块。
 - 公平锁。
 - 注意性能方面。

## Java 中导致饥饿的原因

在 Java 中，下面三个常见的原因会导致线程饥饿：


1. 高优先级线程吞噬所有的低优先级线程的 CPU 时间。
2. 线程被永久堵塞在一个等待进入同步块的状态，因为其他线程总是能在它之前持续地对该同步块进行访问。
3. 线程在等待一个本身(在其上调用 wait())也处于永久等待完成的对象，因为其他线程总是被持续地获得唤醒。

## 高优先级线程吞噬所有的低优先级线程的 CPU 时间

你能为每个线程设置独自的线程优先级，优先级越高的线程获得的 CPU 时间越多，线程优先级值设置在 1 到 10 之间，而这些优先级值所表示行为的准确解释则依赖于你的应用运行平台。对大多数应用来说，你最好是不要改变其优先级值。

## 线程被永久堵塞在一个等待进入同步块的状态

Java 的同步代码区也是一个导致饥饿的因素。Java 的同步代码区对哪个线程允许进入的次序没有任何保障。这就意味着理论上存在一个试图进入该同步区的线程处于被永久堵塞的风险，因为其他线程总是能持续地先于它获得访问，这即是“饥饿”问题，而一个线程被“饥饿致死”正是因为它得不到 CPU 运行时间的机会。

## 线程在等待一个本身(在其上调用 wait())也处于永久等待完成的对象

如果多个线程处在 wait()方法执行上，而对其调用 notify()不会保证哪一个线程会获得唤醒，任何线程都有可能处于继续等待的状态。因此存在这样一个风险：一个等待线程从来得不到唤醒，因为其他等待线程总是能被获得唤醒。

## 在 Java 中实现公平性

虽 Java 不可能实现 100% 的公平性，我们依然可以通过同步结构在线程间实现公平性的提高。

首先来学习一段简单的同步态代码：

```
public class Synchronizer{

    public synchronized void doSynchronized(){

    //do a lot of work which takes a long time

    }
}
```

如果有一个以上的线程调用 doSynchronized()方法，在第一个获得访问的线程未完成前，其他线程将一直处于阻塞状态，而且在这种多线程被阻塞的场景下，接下来将是哪个线程获得访问是没有保障的。

## 使用锁方式替代同步块

为了提高等待线程的公平性，我们使用锁方式来替代同步块。

```
public class Synchronizer{
    Lock lock = new Lock();
    public void doSynchronized() throws InterruptedException{
        this.lock.lock();
        //critical section, do a lot of work which takes a long time
        this.lock.unlock();
    }
}
```

注意到 doSynchronized()不再声明为 synchronized，而是用 lock.lock()和 lock.unlock()来替代。

下面是用 Lock 类做的一个实现：

```
public class Lock{

    private boolean isLocked      = false;

    private Thread lockingThread = null;

    public synchronized void lock() throws InterruptedException{

    while(isLocked){

        wait();

    }

    isLocked = true;

    lockingThread = Thread.currentThread();

}

public synchronized void unlock(){

    if(this.lockingThread != Thread.currentThread()){

         throw new IllegalMonitorStateException(

              "Calling thread has not locked this lock");

         }

    isLocked = false;

    lockingThread = null;

    notify();

    }
}
```

注意到上面对 Lock 的实现，如果存在多线程并发访问 lock()，这些线程将阻塞在对 lock()方法的访问上。另外，如果锁已经锁上（校对注：这里指的是 isLocked 等于 true 时），这些线程将阻塞在 while(isLocked)循环的 wait()调用里面。要记住的是，当线程正在等待进入 lock() 时，可以调用 wait()释放其锁实例对应的同步锁，使得其他多个线程可以进入 lock()方法，并调用 wait()方法。

这回看下 doSynchronized()，你会注意到在 lock()和 unlock()之间的注释：在这两个调用之间的代码将运行很长一段时间。进一步设想，这段代码将长时间运行，和进入 lock()并调用 wait()来比较的话。这意味着大部分时间用在等待进入锁和进入临界区的过程是用在 wait()的等待中，而不是被阻塞在试图进入 lock()方法中。

在早些时候提到过，同步块不会对等待进入的多个线程谁能获得访问做任何保障，同样当调用 notify()时，wait()也不会做保障一定能唤醒线程（至于为什么，请看[线程通信](thread-communication.md)）。因此这个版本的 Lock 类和 doSynchronized()那个版本就保障公平性而言，没有任何区别。

但我们能改变这种情况。当前的 Lock 类版本调用自己的 wait()方法，如果每个线程在不同的对象上调用 wait()，那么只有一个线程会在该对象上调用 wait()，Lock 类可以决定哪个对象能对其调用 notify()，因此能做到有效的选择唤醒哪个线程。

## 公平锁

下面来讲述将上面 Lock 类转变为公平锁 FairLock。你会注意到新的实现和之前的 Lock 类中的同步和 wait()/notify()稍有不同。

准确地说如何从之前的 Lock 类做到公平锁的设计是一个渐进设计的过程，每一步都是在解决上一步的问题而前进的：Nested Monitor Lockout, Slipped Conditions 和 Missed Signals。这些本身的讨论虽已超出本文的范围，但其中每一步的内容都将会专题进行讨论。重要的是，每一个调用 lock()的线程都会进入一个队列，当解锁后，只有队列里的第一个线程被允许锁住 Farlock 实例，所有其它的线程都将处于等待状态，直到他们处于队列头部。

```
public class FairLock {
    private boolean           isLocked       = false;
    private Thread            lockingThread  = null;
    private List<QueueObject> waitingThreads =
            new ArrayList<QueueObject>();

  public void lock() throws InterruptedException{
    QueueObject queueObject           = new QueueObject();
    boolean     isLockedForThisThread = true;
    synchronized(this){
        waitingThreads.add(queueObject);
    }

    while(isLockedForThisThread){
      synchronized(this){
        isLockedForThisThread =
            isLocked || waitingThreads.get(0) != queueObject;
        if(!isLockedForThisThread){
          isLocked = true;
           waitingThreads.remove(queueObject);
           lockingThread = Thread.currentThread();
           return;
         }
      }
      try{
        queueObject.doWait();
      }catch(InterruptedException e){
        synchronized(this) { waitingThreads.remove(queueObject); }
        throw e;
      }
    }
  }

  public synchronized void unlock(){
    if(this.lockingThread != Thread.currentThread()){
      throw new IllegalMonitorStateException(
        "Calling thread has not locked this lock");
    }
    isLocked      = false;
    lockingThread = null;
    if(waitingThreads.size() > 0){
      waitingThreads.get(0).doNotify();
    }
  }
}
``` 

```
public class QueueObject {

    private boolean isNotified = false;

    public synchronized void doWait() throws InterruptedException {

    while(!isNotified){
        this.wait();
    }

    this.isNotified = false;

}

public synchronized void doNotify() {
    this.isNotified = true;
    this.notify();
}

public boolean equals(Object o) {
    return this == o;
}

}
```

首先注意到 lock()方法不在声明为 synchronized，取而代之的是对必需同步的代码，在 synchronized 中进行嵌套。

FairLock 新创建了一个 QueueObject 的实例，并对每个调用 lock()的线程进行入队列。调用 unlock()的线程将从队列头部获取 QueueObject，并对其调用 doNotify()，以唤醒在该对象上等待的线程。通过这种方式，在同一时间仅有一个等待线程获得唤醒，而不是所有的等待线程。这也是实现 FairLock 公平性的核心所在。

请注意，在同一个同步块中，锁状态依然被检查和设置，以避免出现滑漏条件。

还需注意到，QueueObject 实际是一个 semaphore。doWait()和 doNotify()方法在 QueueObject 中保存着信号。这样做以避免一个线程在调用 queueObject.doWait()之前被另一个调用 unlock()并随之调用 queueObject.doNotify()的线程重入，从而导致信号丢失。queueObject.doWait()调用放置在 synchronized(this)块之外，以避免被 monitor 嵌套锁死，所以另外的线程可以解锁，只要当没有线程在 lock 方法的 synchronized(this)块中执行即可。

最后，注意到 queueObject.doWait()在 try – catch 块中是怎样调用的。在 InterruptedException 抛出的情况下，线程得以离开 lock()，并需让它从队列中移除。

## 性能考虑

如果比较 Lock 和 FairLock 类，你会注意到在 FairLock 类中 lock()和 unlock()还有更多需要深入的地方。这些额外的代码会导致 FairLock 的同步机制实现比 Lock 要稍微慢些。究竟存在多少影响，还依赖于应用在 FairLock 临界区执行的时长。执行时长越大，FairLock 带来的负担影响就越小，当然这也和代码执行的频繁度相关。
