# 信号量

Semaphore（信号量）是一个线程同步结构，用于在线程间传递信号，以避免出现信号丢失（译者注：下文会具体介绍），或者像锁一样用于保护一个关键区域。自从 5.0 开始，jdk 在 java.util.concurrent 包里提供了 Semaphore 的官方实现，因此大家不需要自己去实现 Semaphore。但是还是很有必要去熟悉如何使用 Semaphore 及其背后的原理

本文的涉及的主题如下：


1. 简单的 Semaphore 实现
2. 使用 Semaphore 来发出信号
3. 可计数的 Semaphore
4. 有上限的 Semaphore
5. 把 Semaphore 当锁来使用

## 简单的 Semaphore 实现

下面是一个信号量的简单实现：

```
public class Semaphore {

private boolean signal = false;

public synchronized void take() {

this.signal = true;

this.notify();

}

public synchronized void release() throws InterruptedException{

while(!this.signal) wait();

this.signal = false;

}

}
```

Take 方法发出一个被存放在 Semaphore 内部的信号，而 Release 方法则等待一个信号，当其接收到信号后，标记位 signal 被清空，然后该方法终止。

使用这个 semaphore 可以避免错失某些信号通知。用 take 方法来代替 notify，release 方法来代替 wait。如果某线程在调用 release 等待之前调用 take 方法，那么调用 release 方法的线程仍然知道 take 方法已经被某个线程调用过了，因为该 Semaphore 内部保存了 take 方法发出的信号。而 wait 和 notify 方法就没有这样的功能。

当用 semaphore 来产生信号时，take 和 release 这两个方法名看起来有点奇怪。这两个名字来源于后面把 semaphore 当做锁的例子，后面会详细介绍这个例子，在该例子中，take 和 release 这两个名字会变得很合理。

## 使用 Semaphore 来产生信号

下面的例子中，两个线程通过 Semaphore 发出的信号来通知对方

```
Semaphore semaphore = new Semaphore();

SendingThread sender = new SendingThread(semaphore)；

ReceivingThread receiver = new ReceivingThread(semaphore);

receiver.start();

sender.start();

public class SendingThread {

Semaphore semaphore = null;

public SendingThread(Semaphore semaphore){

this.semaphore = semaphore;

}

public void run(){

while(true){

//do something, then signal

this.semaphore.take();

}

}

}

public class RecevingThread {

Semaphore semaphore = null;

public ReceivingThread(Semaphore semaphore){

this.semaphore = semaphore;

}

public void run(){

while(true){

this.semaphore.release();

//receive signal, then do something...

}

}

}
```

## 可计数的 Semaphore

上面提到的 Semaphore 的简单实现并没有计算通过调用 take 方法所产生信号的数量。可以把它改造成具有计数功能的 Semaphore。下面是一个可计数的 Semaphore 的简单实现。

```
public class CountingSemaphore {

private int signals = 0;

public synchronized void take() {

this.signals++;

this.notify();

}

public synchronized void release() throws InterruptedException{

while(this.signals == 0) wait();

this.signals--;

}

}
```

## 有上限的 Semaphore

上面的 CountingSemaphore 并没有限制信号的数量。下面的代码将 CountingSemaphore 改造成一个信号数量有上限的 BoundedSemaphore。

```
public class BoundedSemaphore {

private int signals = 0;

private int bound   = 0;

public BoundedSemaphore(int upperBound){

this.bound = upperBound;

}

public synchronized void take() throws InterruptedException{

while(this.signals == bound) wait();

this.signals++;

this.notify();

}

public synchronized void release() throws InterruptedException{

while(this.signals == 0) wait();

this.signals--;

this.notify();

}

}
```

在 BoundedSemaphore 中，当已经产生的信号数量达到了上限，take 方法将阻塞新的信号产生请求，直到某个线程调用 release 方法后，被阻塞于 take 方法的线程才能传递自己的信号。

## 把 Semaphore 当锁来使用

当信号量的数量上限是 1 时，Semaphore 可以被当做锁来使用。通过 take 和 release 方法来保护关键区域。请看下面的例子：

```
BoundedSemaphore semaphore = new BoundedSemaphore(1);

...

semaphore.take();

try{

//critical section

} finally {

semaphore.release();

}
```

在前面的例子中，Semaphore 被用来在多个线程之间传递信号，这种情况下，take 和 release 分别被不同的线程调用。但是在锁这个例子中，take 和 release 方法将被同一线程调用，因为只允许一个线程来获取信号（允许进入关键区域的信号），其它调用 take 方法获取信号的线程将被阻塞，知道第一个调用 take 方法的线程调用 release 方法来释放信号。对 release 方法的调用永远不会被阻塞，这是因为任何一个线程都是先调用 take 方法，然后再调用 release。

通过有上限的 Semaphore 可以限制进入某代码块的线程数量。设想一下，在上面的例子中，如果 BoundedSemaphore 上限设为 5 将会发生什么？意味着允许 5 个线程同时访问关键区域，但是你必须保证，这个 5 个线程不会互相冲突。否则你的应用程序将不能正常运行。

必须注意，release 方法应当在 finally 块中被执行。这样可以保在关键区域的代码抛出异常的情况下，信号也一定会被释放。
