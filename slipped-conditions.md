# Slipped Conditions

所谓 Slipped conditions，就是说， 从一个线程检查某一特定条件到该线程操作此条件期间，这个条件已经被其它线程改变，导致第一个线程在该条件上执行了错误的操作。这里有一个简单的例子：

```
public class Lock {
    private boolean isLocked = true;

    public void lock(){
      synchronized(this){
        while(isLocked){
          try{
            this.wait();
          } catch(InterruptedException e){
            //do nothing, keep waiting
          }
        }
      }

      synchronized(this){
        isLocked = true;
      }
    }

    public synchronized void unlock(){
      isLocked = false;
      this.notify();
    }
}
```

我们可以看到，lock()方法包含了两个同步块。第一个同步块执行 wait 操作直到 isLocked 变为 false 才退出，第二个同步块将 isLocked 置为 true，以此来锁住这个 Lock 实例避免其它线程通过 lock()方法。

我们可以设想一下，假如在某个时刻 isLocked 为 false， 这个时候，有两个线程同时访问 lock 方法。如果第一个线程先进入第一个同步块，这个时候它会发现 isLocked 为 false，若此时允许第二个线程执行，它也进入第一个同步块，同样发现 isLocked 是 false。现在两个线程都检查了这个条件为 false，然后它们都会继续进入第二个同步块中并设置 isLocked 为 true。

这个场景就是 slipped conditions 的例子，两个线程检查同一个条件， 然后退出同步块，因此在这两个线程改变条件之前，就允许其它线程来检查这个条件。换句话说，条件被某个线程检查到该条件被此线程改变期间，这个条件已经被其它线程改变过了。

为避免 slipped conditions，条件的检查与设置必须是原子的，也就是说，在第一个线程检查和设置条件期间，不会有其它线程检查这个条件。

解决上面问题的方法很简单，只是简单的把 isLocked = true 这行代码移到第一个同步块中，放在 while 循环后面即可：

```
public class Lock {
    private boolean isLocked = true;

    public void lock(){
      synchronized(this){
        while(isLocked){
          try{
            this.wait();
          } catch(InterruptedException e){
            //do nothing, keep waiting
          }
        }
        isLocked = true;
      }
    }

    public synchronized void unlock(){
      isLocked = false;
      this.notify();
    }
}
```

现在检查和设置 isLocked 条件是在同一个同步块中原子地执行了。

## 一个更现实的例子

也许你会说，我才不可能写这么挫的代码，还觉得 slipped conditions 是个相当理论的问题。但是第一个简单的例子只是用来更好的展示 slipped conditions。

[饥饿和公平](starvation-and-fairness.md)中实现的公平锁也许是个更现实的例子。再看下[嵌套管程锁死](nested-monitor-lockout.md)中那个幼稚的实现，如果我们试图解决其中的嵌套管程锁死问题，很容易产生 slipped conditions 问题。 首先让我们看下[嵌套管程锁死](nested-monitor-lockout.md)中的例子：

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

      while(isLocked || waitingThreads.get(0) != queueObject){

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
    isLocked      = false;
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

我们可以看到 synchronized(queueObject)及其中的 queueObject.wait()调用是嵌在 synchronized(this)块里面的，这会导致嵌套管程锁死问题。为避免这个问题，我们必须将 synchronized(queueObject)块移出 synchronized(this)块。移出来之后的代码可能是这样的：

```
//Fair Lock implementation with slipped conditions problem
public class FairLock {
  private boolean isLocked = false;
  private Thread lockingThread  = null;
  private List waitingThreads =
            new ArrayList();

  public void lock() throws InterruptedException{
    QueueObject queueObject = new QueueObject();

    synchronized(this){
      waitingThreads.add(queueObject);
    }

    boolean mustWait = true;
    while(mustWait){

      synchronized(this){
        mustWait = isLocked || waitingThreads.get(0) != queueObject;
      }

      synchronized(queueObject){
        if(mustWait){
          try{
            queueObject.wait();
          }catch(InterruptedException e){
            waitingThreads.remove(queueObject);
            throw e;
          }
        }
      }
    }

    synchronized(this){
      waitingThreads.remove(queueObject);
      isLocked = true;
      lockingThread = Thread.currentThread();
    }
  }
}
```

注意：因为我只改动了 lock()方法，这里只展现了 lock 方法。

现在 lock()方法包含了 3 个同步块。

第一个，synchronized(this)块通过 mustWait = isLocked || waitingThreads.get(0) != queueObject 检查内部变量的值。

第二个，synchronized(queueObject)块检查线程是否需要等待。也有可能其它线程在这个时候已经解锁了，但我们暂时不考虑这个问题。我们就假设这个锁处在解锁状态，所以线程会立马退出 synchronized(queueObject)块。

第三个，synchronized(this)块只会在 mustWait 为 false 的时候执行。它将 isLocked 重新设回 true，然后离开 lock()方法。

设想一下，在锁处于解锁状态时，如果有两个线程同时调用 lock()方法会发生什么。首先，线程 1 会检查到 isLocked 为 false，然后线程 2 同样检查到 isLocked 为 false。接着，它们都不会等待，都会去设置 isLocked 为 true。这就是 slipped conditions 的一个最好的例子。

## 解决 Slipped Conditions 问题

要解决上面例子中的 slipped conditions 问题，最后一个 synchronized(this)块中的代码必须向上移到第一个同步块中。为适应这种变动，代码需要做点小改动。下面是改动过的代码：

```
//Fair Lock implementation without nested monitor lockout problem,
//but with missed signals problem.
public class FairLock {
  private boolean isLocked = false;
  private Thread lockingThread  = null;
  private List waitingThreads =
            new ArrayList();

  public void lock() throws InterruptedException{
    QueueObject queueObject = new QueueObject();

    synchronized(this){
      waitingThreads.add(queueObject);
    }

    boolean mustWait = true;
    while(mustWait){
      synchronized(this){
        mustWait = isLocked || waitingThreads.get(0) != queueObject;
        if(!mustWait){
          waitingThreads.remove(queueObject);
          isLocked = true;
          lockingThread = Thread.currentThread();
          return;
        }
      }     

      synchronized(queueObject){
        if(mustWait){
          try{
            queueObject.wait();
          }catch(InterruptedException e){
            waitingThreads.remove(queueObject);
            throw e;
          }
        }
      }
    }
  }
}
```

我们可以看到对局部变量 mustWait 的检查与赋值是在同一个同步块中完成的。还可以看到，即使在 synchronized(this)块外面检查了 mustWait，在 while(mustWait)子句中，mustWait 变量从来没有在 synchronized(this)同步块外被赋值。当一个线程检查到 mustWait 是 false 的时候，它将自动设置内部的条件（isLocked），所以其它线程再来检查这个条件的时候，它们就会发现这个条件的值现在为 true 了。

synchronized(this)块中的 return;语句不是必须的。这只是个小小的优化。如果一个线程肯定不会等待（即 mustWait 为 false），那么就没必要让它进入到 synchronized(queueObject)同步块中和执行 if(mustWait)子句了。

细心的读者可能会注意到上面的公平锁实现仍然有可能丢失信号。设想一下，当该 FairLock 实例处于锁定状态时，有个线程来调用 lock()方法。执行完第一个 synchronized(this)块后，mustWait 变量的值为 true。再设想一下调用 lock()的线程是通过抢占式的，拥有锁的那个线程那个线程此时调用了 unlock()方法，但是看下之前的 unlock()的实现你会发现，它调用了 queueObject.notify()。但是，因为 lock()中的线程还没有来得及调用 queueObject.wait()，所以 queueObject.notify()调用也就没有作用了，信号就丢失掉了。如果调用 lock()的线程在另一个线程调用 queueObject.notify()之后调用 queueObject.wait()，这个线程会一直阻塞到其它线程调用 unlock 方法为止，但这永远也不会发生。

公平锁实现的信号丢失问题在[饥饿和公平](starvation-and-fairness.md)一文中我们已有过讨论，把 QueueObject 转变成一个信号量，并提供两个方法：doWait()和 doNotify()。这些方法会在 QueueObject 内部对信号进行存储和响应。用这种方式，即使 doNotify()在 doWait()之前调用，信号也不会丢失。
