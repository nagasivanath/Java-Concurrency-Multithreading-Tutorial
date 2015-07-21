# CAS

CAS（Compare and swap）比较和替换是设计并发算法时用到的一种技术。简单来说，比较和替换是使用一个期望值和一个变量的当前值进行比较，如果当前变量的值与我们期望的值相等，就使用一个新值替换当前变量的值。这听起来可能有一点复杂但是实际上你理解之后发现很简单，接下来，让我们跟深入的了解一下这项技术。


## CAS 的使用场景

在程序和算法中一个经常出现的模式就是“check and act”模式。先检查后操作模式发生在代码中首先检查一个变量的值，然后再基于这个值做一些操作。下面是一个简单的示例：

```
class MyLock {

    private boolean locked = false;

    public boolean lock() {
        if(!locked) {
            locked = true;
            return true;
        }
        return false;
    }
}
```

上面这段代码，如果用在多线程的程序会出现很多错误，不过现在请忘掉它。

如你所见，lock()方法首先检查 locked>成员变量是否等于 false，如果等于，就将 locked 设为 true。

如果同个线程访问同一个 MyLock 实例，上面的 lock()将不能保证正常工作。如果一个线程检查 locked 的值，然后将其设置为 false，与此同时，一个线程 B 也在检查 locked 的值，又或者，在线程 A 将 locked 的值设为 false 之前。因此，线程 A 和线程 B 可能都看到 locked 的值为 false，然后两者都基于这个信息做一些操作。

为了在一个多线程程序中良好的工作，”check then act” 操作必须是原子的。原子就是说”check“操作和”act“被当做一个原子代码块执行。不存在多个线程同时执行原子块。

下面是一个代码示例，把之前的 lock()方法用 synchronized 关键字重构成一个原子块。

```
class MyLock {

    private boolean locked = false;

    public synchronized boolean lock() {
        if(!locked) {
            locked = true;
            return true;
        }
        return false;
    }
}
```

现在 lock()方法是同步的，所以，在某一时刻只能有一个线程在同一个 MyLock 实例上执行它。

原子的 lock 方法实际上是一个”compare and swap“的例子。

## CAS 用作原子操作

现在 CPU 内部已经执行原子的 CAS 操作。Java5 以来，你可以使用 java.util.concurrent.atomic 包中的一些原子类来使用 CPU 中的这些功能。

下面是一个使用 AtomicBoolean 类实现 lock()方法的例子：

```
public static class MyLock {
    private AtomicBoolean locked = new AtomicBoolean(false);

    public boolean lock() {
        return locked.compareAndSet(false, true);
    }

}
```

locked 变量不再是 boolean 类型而是 AtomicBoolean。这个类中有一个 compareAndSet()方法，它使用一个期望值和 AtomicBoolean 实例的值比较，和两者相等，则使用一个新值替换原来的值。在这个例子中，它比较 locked 的值和 false，如果 locked 的值为 false，则把修改为 true。
如果值被替换了，compareAndSet()返回 true，否则，返回 false。

使用 Java5+提供的 CAS 特性而不是使用自己实现的的好处是 Java5+中内置的 CAS 特性可以让你利用底层的你的程序所运行机器的 CPU 的 CAS 特性。这会使还有 CAS 的代码运行更快。
