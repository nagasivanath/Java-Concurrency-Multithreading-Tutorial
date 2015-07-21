# 嵌套管程锁死

嵌套管程锁死类似于死锁， 下面是一个嵌套管程锁死的场景：

> 线程 1 获得 A 对象的锁。  
> 线程 1 获得对象 B 的锁（同时持有对象 A 的锁）。  
> 线程 1 决定等待另一个线程的信号再继续。  
> 线程 1 调用 B.wait()，从而释放了 B 对象上的锁，但仍然持有对象 A 的锁。  
>   
> 线程 2 需要同时持有对象 A 和对象 B 的锁，才能向线程 1 发信号。  
> 线程 2 无法获得对象 A 上的锁，因为对象 A 上的锁当前正被线程 1 持有。  
> 线程 2 一直被阻塞，等待线程 1 释放对象 A 上的锁。  
>   
> 线程 1 一直阻塞，等待线程 2 的信号，因此，不会释放对象 A 上的锁，  
>    而线程 2 需要对象 A 上的锁才能给线程 1 发信号……  

你可以能会说，这是个空想的场景，好吧，让我们来看看下面这个比较挫的 Lock 实现：

```
//lock implementation with nested monitor lockout problem
public class Lock{
    protected MonitorObject monitorObject = new MonitorObject();
    protected boolean isLocked = false;

    public void lock() throws InterruptedException{
        synchronized(this){
            while(isLocked){
                synchronized(this.monitorObject){
                    this.monitorObject.wait();
                }
            }
            isLocked = true;
        }
    }

    public void unlock(){
        synchronized(this){
            this.isLocked = false;
            synchronized(this.monitorObject){
                this.monitorObject.notify();
            }
        }
    }
}
```

可以看到，lock()方法首先在”this”上同步，然后在 monitorObject 上同步。如果 isLocked 等于 false，因为线程不会继续调用 monitorObject.wait()，那么一切都没有问题 。但是如果 isLocked 等于 true，调用 lock()方法的线程会在 monitorObject.wait()上阻塞。

这里的问题在于，调用 monitorObject.wait()方法只释放了 monitorObject 上的管程对象，而与”this“关联的管程对象并没有释放。换句话说，这个刚被阻塞的线程仍然持有”this”上的锁。

*（校对注：如果一个线程持有这种 Lock 的时候另一个线程执行了 lock 操作）*当一个已经持有这种 Lock 的线程想调用 unlock()，就会在 unlock()方法进入 synchronized(this)块时阻塞。这会一直阻塞到在 lock()方法中等待的线程离开 synchronized(this)块。但是，在 unlock 中 isLocked 变为 false，monitorObject.notify()被执行之后，lock()中等待的线程才会离开 synchronized(this)块。

简而言之，在 lock 方法中等待的线程需要其它线程成功调用 unlock 方法来退出 lock 方法，但是，在 lock()方法离开外层同步块之前，没有线程能成功执行 unlock()。

结果就是，任何调用 lock 方法或 unlock 方法的线程都会一直阻塞。这就是嵌套管程锁死。

## 一个更现实的例子

你可能会说，这么挫的实现方式我怎么可能会做呢？你或许不会在里层的管程对象上调用 wait 或 notify 方法，但完全有可能会在外层的 this 上调。
有很多类似上面例子的情况。例如，如果你准备实现一个公平锁。你可能希望每个线程在它们各自的 QueueObject 上调用 wait()，这样就可以每次唤醒一个线程。

下面是一个比较挫的公平锁实现方式：

```
//Fair Lock implementation with nested monitor lockout problem
public class FairLock {
    private boolean isLocked = false;
    private Thread lockingThread = null;
    private List waitingThreads =
        new ArrayList();

    public void lock() throws InterruptedException{
        QueueObject queueObject = new QueueObject();

        synchronized(this){
            waitingThreads.add(queueObject);

            while(isLocked ||
                waitingThreads.get(0) != queueObject){

                synchronized(queueObject){
                    try{
                        queueObject.wait();
                    }catch(InterruptedException e){
                        waitingThreads.remove(queueObject);
                        throw e;
                    }
                }
            }
            waitingThreads.remove(queueObject);
            isLocked = true;
            lockingThread = Thread.currentThread();
        }
    }

    public synchronized void unlock(){
        if(this.lockingThread != Thread.currentThread()){
            throw new IllegalMonitorStateException(
                "Calling thread has not locked this lock");
        }
        isLocked = false;
        lockingThread = null;
        if(waitingThreads.size() > 0){
            QueueObject queueObject = waitingThread.get(0);
            synchronized(queueObject){
                queueObject.notify();
            }
        }
    }
}
public class QueueObject {}
```

乍看之下，嗯，很好，但是请注意 lock 方法是怎么调用 queueObject.wait()的，在方法内部有两个 synchronized 块，一个锁定 this，一个嵌在上一个 synchronized 块内部，它锁定的是局部变量 queueObject。

当一个线程调用 queueObject.wait()方法的时候，它仅仅释放的是在 queueObject 对象实例的锁，并没有释放”this”上面的锁。

现在我们还有一个地方需要特别注意， unlock 方法被声明成了 synchronized，这就相当于一个 synchronized（this）块。这就意味着，如果一个线程在 lock()中等待，该线程将持有与 this 关联的管程对象。所有调用 unlock()的线程将会一直保持阻塞，等待着前面那个已经获得 this 锁的线程释放 this 锁，但这永远也发生不了，因为只有某个线程成功地给 lock()中等待的线程发送了信号，this 上的锁才会释放，但只有执行 unlock()方法才会发送这个信号。

因此，上面的公平锁的实现会导致嵌套管程锁死。更好的公平锁实现方式可以参考 Starvation and Fairness。

## 嵌套管程锁死 VS 死锁

嵌套管程锁死与死锁很像：都是线程最后被一直阻塞着互相等待。

但是两者又不完全相同。在[死锁](deadlock.md) 中我们已经对死锁有了个大概的解释，死锁通常是因为两个线程获取锁的顺序不一致造成的，线程 1 锁住 A，等待获取 B，线程 2 已经获取了 B，再等待获取 A。如[避免死锁](deadlock-prevention.md)中所说的，死锁可以通过总是以相同的顺序获取锁来避免。

但是发生嵌套管程锁死时锁获取的顺序是一致的。线程 1 获得 A 和 B，然后释放 B，等待线程 2 的信号。线程 2 需要同时获得 A 和 B，才能向线程 1 发送信号。所以，一个线程在等待唤醒，另一个线程在等待想要的锁被释放。

不同点归纳如下：

> 死锁中，二个线程都在等待对方释放锁。  
>   
> 嵌套管程锁死中，线程 1 持有锁 A，同时等待从线程 2 发来的信号，线程 2 需要锁 A 来发信号给线程 1。  