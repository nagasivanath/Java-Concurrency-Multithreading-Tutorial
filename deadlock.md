# 死锁

死锁是两个或更多线程阻塞着等待其它处于死锁状态的线程所持有的锁。死锁通常发生在多个线程同时但以不同的顺序请求同一组锁的时候。

例如，如果线程 1 锁住了 A，然后尝试对 B 进行加锁，同时线程 2 已经锁住了 B，接着尝试对 A 进行加锁，这时死锁就发生了。线程 1 永远得不到 B，线程 2 也永远得不到 A，并且它们永远也不会知道发生了这样的事情。为了得到彼此的对象（A 和 B），它们将永远阻塞下去。这种情况就是一个死锁。


该情况如下：

```
Thread 1  locks A, waits for B
Thread 2  locks B, waits for A
```

这里有一个 TreeNode 类的例子，它调用了不同实例的 synchronized 方法：

```
public class TreeNode {
    TreeNode parent   = null;  
    List children = new ArrayList();

    public synchronized void addChild(TreeNode child){
        if(!this.children.contains(child)) {
            this.children.add(child);
            child.setParentOnly(this);
        }
    }
  
    public synchronized void addChildOnly(TreeNode child){
        if(!this.children.contains(child){
            this.children.add(child);
        }
    }
  
    public synchronized void setParent(TreeNode parent){
        this.parent = parent;
        parent.addChildOnly(this);
    }

    public synchronized void setParentOnly(TreeNode parent){
        this.parent = parent;
    }
}
```

如果线程 1 调用 parent.addChild(child)方法的同时有另外一个线程 2 调用 child.setParent(parent)方法，两个线程中的 parent 表示的是同一个对象，child 亦然，此时就会发生死锁。下面的伪代码说明了这个过程：

```
Thread 1: parent.addChild(child); //locks parent
          --> child.setParentOnly(parent);

Thread 2: child.setParent(parent); //locks child
          --> parent.addChildOnly()
```

首先线程 1 调用 parent.addChild(child)。因为 addChild()是同步的，所以线程 1 会对 parent 对象加锁以不让其它线程访问该对象。

然后线程 2 调用 child.setParent(parent)。因为 setParent()是同步的，所以线程 2 会对 child 对象加锁以不让其它线程访问该对象。

现在 child 和 parent 对象被两个不同的线程锁住了。接下来线程 1 尝试调用 child.setParentOnly()方法，但是由于 child 对象现在被线程 2 锁住的，所以该调用会被阻塞。线程 2 也尝试调用 parent.addChildOnly()，但是由于 parent 对象现在被线程 1 锁住，导致线程 2 也阻塞在该方法处。现在两个线程都被阻塞并等待着获取另外一个线程所持有的锁。

注意：像上文描述的，这两个线程需要同时调用 parent.addChild(child)和 child.setParent(parent)方法，并且是同一个 parent 对象和同一个 child 对象，才有可能发生死锁。上面的代码可能运行一段时间才会出现死锁。

这些线程需要**同时**获得锁。举个例子，如果线程 1 稍微领先线程 2，然后成功地锁住了 A 和 B 两个对象，那么线程 2 就会在尝试对 B 加锁的时候被阻塞，这样死锁就不会发生。因为线程调度通常是不可预测的，因此没有一个办法可以准确预测**什么时候**死锁会发生，仅仅是 **可能会**发生。

## 更复杂的死锁

死锁可能不止包含 2 个线程，这让检测死锁变得更加困难。下面是 4 个线程发生死锁的例子：

```
Thread 1  locks A, waits for B
Thread 2  locks B, waits for C
Thread 3  locks C, waits for D
Thread 4  locks D, waits for A
```

线程 1 等待线程 2，线程 2 等待线程 3，线程 3 等待线程 4，线程 4 等待线程 1。

## 数据库的死锁

更加复杂的死锁场景发生在数据库事务中。一个数据库事务可能由多条 SQL 更新请求组成。当在一个事务中更新一条记录，这条记录就会被锁住避免其他事务的更新请求，直到第一个事务结束。同一个事务中每一个更新请求都可能会锁住一些记录。

当多个事务同时需要对一些相同的记录做更新操作时，就很有可能发生死锁，例如：

```
Transaction 1, request 1, locks record 1 for update
Transaction 2, request 1, locks record 2 for update
Transaction 1, request 2, tries to lock record 2 for update.
Transaction 2, request 2, tries to lock record 1 for update.
```

因为锁发生在不同的请求中，并且对于一个事务来说不可能提前知道所有它需要的锁，因此很难检测和避免数据库事务中的死锁。
