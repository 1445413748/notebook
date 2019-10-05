### AQS 共享锁

下面用 Semaphore 为例

#### 加锁过程

```java
/* Semaphore.acquire */
// 获取信号量，也就是加锁
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}


/* AQS.acquireSharedInterruptibly, 类似有acquireShared，只是没有中断检测*/
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
	// 尝试获取共享锁    
    if (tryAcquireShared(arg) < 0)
        // 如果 tryAcquireShared 返回负数，则尝试获取锁失败，开始不断获取锁或阻塞唤醒后继续尝试获取
        doAcquireSharedInterruptibly(arg);
}


/* Semaphore.tryAcquireShared， 这里以公平锁为例*/
// 尝试获取共享锁
protected int tryAcquireShared(int acquires) {
    for (;;) {
        // 如果队列中还有阻塞的队列，则获取共享锁失败
        // 非公平锁的区别就是没有这个检测
        if (hasQueuedPredecessors())
            return -1;
        // 下面是获取 state 并更新
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            // 在 Semaphore 中是当 state 大于0时可以获取锁，否则会被阻塞
            // 所以 remaining 大于等于0 说明线程获取锁成功，小于0 说明获取锁失败
            return remaining;
    }
}


/* AQS.doAcquireSharedInterruptibly, 和doAcquireShared类似*/
// 如果尝试获取共享锁失败，也就是tryAcquireShared(arg) < 0进入这个方法
// 在这个方法中一直尝试获取共享锁
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 将当前线程包装成 Node 节点并加入队列
    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            // 得到当前线程节点的前驱节点
            final Node p = node.predecessor();
            // 如果前驱节点是阻塞队列的头结点就继续尝试获取共享锁
            if (p == head) {
                int r = tryAcquireShared(arg);
                // r>=0 说明了获取共享锁成功
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    // 回收旧的头结点，看 setHeadAndPropagate 的图
                    p.next = null; // help GC
                    return;
                }
            }
            // 来到这里说明要不就是 node 前驱节点不是 head，要不就是获取共享锁失败
            // 于是尝试将线程阻塞，注意唤醒后也是从这里继续执行
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}


/* AQS.setHeadAndPropagate */
// 如果 doAcquireSharedInterruptibly 中获取锁成功，则会进入这个方法
// 这个方法主要就是更新阻塞队列的头结点，如果当前节点有后继节点，唤醒后继节点
private void setHeadAndPropagate(Node node, int propagate) {
    //setHeadAndPropagate(node, r); r == propagate >= 0
    // 保存原先节点
    Node h = head;
    
    /* 设置头结点之前
         +------+         +-----+         +-----+
    Head |      | <---->  |node | <---->  |     |  tail
         +------+         +-----+         +-----+
    */
    
    // 设置 node 为新头结点
    setHead(node);
    
    /*
     设置新头节点之后
            +------+    head +-----+         +-----+
    oldHead |      |  ---->  |node | <---->  |     |  tail
            +------+         +-----+         +-----+
    */
    
    // propagate > 0 ，也就是 r > 0，说明了还是有剩下的共享锁可以被获取，所以执行唤醒操作
    // 那这里为什么还有这么多其他的判断条件
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // s 为空或者 s 是获取的是共享锁
        if (s == null || s.isShared())
            // 唤醒操作
            doReleaseShared();
    }
}



/* AQS.doReleaseShared */
// 唤醒线程或者设置传播状态
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        // 如果阻塞队列有后继节点，执行唤醒
        if (h != null && h != tail) {
            // 得到头结点的状态
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                // 成功将头结点的状态设置为0，才唤醒线程
                if (!h.compareAndSetWaitStatus(Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 如果头结点状态为0，则将头结点状态设置为传播状态
            // 在setHeadAndPropagate中调用了doReleaseShared方法，说明
            // 还有线程需要被唤醒，如果不设置传播状态，因为 ws == 0，该线程
            // 将一直阻塞，队列死亡。
            else if (ws == 0 &&
                     // Node.PROPAGATE = -3
                     !h.compareAndSetWaitStatus(0, Node.PROPAGATE)) 
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}

```





#### 解锁过程

```Java
/* Semaphore.release */
public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}


/* AQS.releaseShared */
public final boolean releaseShared(int arg) {
    // 先尝试释放共享锁
    if (tryReleaseShared(arg)) {
        // 如果尝试释放共享锁失败，则一直尝试释放
        doReleaseShared();
        return true;
    }
    return false;
}


/* Semaphore.tryReleaseShared */
// 尝试释放锁
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
        if (compareAndSetState(current, next))
            return true;
    }
}
```



---

**setHeadAndPropagate 中为什么有还有这么多其他判断?**

**Node 节点中的 Propagate 状态存在的意义是什么？**

```java
private void setHeadAndPropagate(Node node, int propagate) {
    //setHeadAndPropagate(node, r); r == propagate >= 0
    // 保存原先节点
    Node h = head;
    
    /* 设置头结点之前
         +------+         +-----+         +-----+
    Head |      | <---->  |node | <---->  |     |  tail
         +------+         +-----+         +-----+
    */
    
    // 设置 node 为新头结点
    setHead(node);
    
    /*
     设置新头节点之后
            +------+    head +-----+         +-----+
    oldHead |      |  ---->  |node | <---->  |     |  tail
            +------+         +-----+         +-----+
    */
    
    // propagate > 0 ，也就是 r > 0，说明了还是有剩下的共享锁可以被获取，所以执行唤醒操作
    // 那这里为什么还有这么多其他的判断条件
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // s 为空或者 s 是获取的是共享锁
        if (s == null || s.isShared())
            // 唤醒操作
            doReleaseShared();
    }
}
```

我们怎么调用上面这个方法的？

```java
            if (p == head) {
                int r = tryAcquireShared(arg);
                // r>=0 说明了获取共享锁成功
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    // 回收旧的头结点，看 setHeadAndPropagate 的图
                    p.next = null; // help GC
                    return;
                }
            }
```

r >= 0 说明头结点后面的第一个节点成功获取锁，**并且如果 r > 0 则说明了还有剩余的共享锁可以获取， r == 0 说明刚好使用完所有的共享锁。**

接着调用 setHeadAndPropagate(node, r)

按道理来说，只要 propagate（也就是传进来的r）大于0（有剩余共享锁），我才去唤醒其他阻塞的线程，那我 setHeadAndPropagate 不是可以做下面的修改

```java
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // s 为空或者 s 是获取的是共享锁
        if (s == null || s.isShared())
            // 唤醒操作
            doReleaseShared();
    }

//  ---->

    if (propagate > 0) {
        Node s = node.next;
        // s 为空或者 s 是获取的是共享锁
        if (s == null || s.isShared())
            // 唤醒操作
            doReleaseShared();
    }
```

但是这样写是存在 bug 的，可能会导致线程 hang 住

如下面的程序

```java
import java.util.concurrent.Semaphore;

public class TestSemaphore {

   private static Semaphore sem = new Semaphore(0);

   private static class Thread1 extends Thread {
       @Override
       public void run() {
           sem.acquireUninterruptibly();
       }
   }

   private static class Thread2 extends Thread {
       @Override
       public void run() {
           sem.release();
       }
   }

   public static void main(String[] args) throws InterruptedException {
       for (int i = 0; i < 10000000; i++) {
           Thread t1 = new Thread1();
           Thread t2 = new Thread1();
           Thread t3 = new Thread2();
           Thread t4 = new Thread2();
           t1.start();
           t2.start();
           t3.start();
           t4.start();
           t1.join();
           t2.join();
           t3.join();
           t4.join();
           System.out.println(i);
       }
   }
}
```

[可在这找到](http://bugs.java.com/view_bug.do?bug_id=6801020)

对于上面的程序，t1, t2 用户获取信号量， t3, t4 用于释放信号量，信号量初始化为0。

假设某个循环中，t1, t2 启动后获取不到信号量被阻塞，如下：

```
                            t1              t2
         +------+         +-----+         +-----+
    head |      | <---->  |     | <---->  |     |  tail
         +------+         +-----+         +-----+
         
```

接下来执行 `t3.join()` ，执行 `releaseShared`  -> `tryReleaseShared` -> `doReleaseShared`。在 `doReleaseShared` 中会唤醒线程 t1，t1 被唤醒前，会将 head 节点的 waitStatus 设置为 0，接着 t1 从 `doAcquireSharedInterruptibly` 中继续执行，也就是走下面的循环

```java
        for (;;) {
            // 得到当前线程节点的前驱节点
            final Node p = node.predecessor();
            // 如果前驱节点是阻塞队列的头结点就继续尝试获取共享锁
            if (p == head) {
                int r = tryAcquireShared(arg);
                // r>=0 说明了获取共享锁成功
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    // 回收旧的头结点，看 setHeadAndPropagate 的图
                    p.next = null; // help GC
                    return;
                }
            }
            // 来到这里说明要不就是 node 前驱节点不是 head，要不就是获取共享锁失败
            // 于是尝试将线程阻塞，注意唤醒后也是从这里继续执行
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
```

时刻一：假设此时 t1 刚执行完 `tryAcquireShared(arg)`，（这是会成功的，**且返回值为0**）就停在这里了，**注意此时 head 节点的 waitStatus 已经被设置为 0**。



时刻二：假设 t4 也释放锁，执行 `releaseShared` -> `tryReleaseShared` ->  `doReleaseShared` ，在 `doReleaseShared` 中因为 head.waitStatus 是 0 （这里的 head 跟 t1 读到的 head 一样，因为此时 t1 还没有设置自己为头结点），所以不会执行 `unparkSuccessor(h)`



时刻三：接下来 t1 继续执行会进入 `setHeadAndPropagate`，注意 `r == 0`，假设在 `setHeadAndPropagate` 中我们仅仅依赖 `propagate > 0` 来判断是否需要唤醒，那么 t1 并不会执行唤醒操作。

```java
    if (propagate > 0) {
        Node s = node.next;
        // s 为空或者 s 是获取的是共享锁
        if (s == null || s.isShared())
            // 唤醒操作
            doReleaseShared();
    }
```

所以，最后线程 t2 根本不会有人去唤醒它。



**那 Node 中 Propagate 状态如何解决这个问题？**

主要在时刻二中，当 t4 读到 head.waitStatus 是 0 后，它会设置 head 的 waitStatus 为 PROPAGATE。因此，到了时刻三中，t1 在 `setHeadAndPropagate` 中可以得到 h.waitStatus < 0，因此执行唤醒操作唤醒 t2。

```java
private void setHeadAndPropagate(Node node, int propagate) {
    //setHeadAndPropagate(node, r); r == propagate >= 0
    Node h = head;
    // 设置 node 为新头结点
    setHead(node);
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // s 为空或者 s 是获取的是共享锁
        if (s == null || s.isShared())
            // 唤醒操作
            doReleaseShared();
    }
}
```



也就是说，上面会产生线程 hang 住 bug 的case在引入 PROPAGATE 后可以被规避掉。在PROPAGATE引入之前，之所以可能会出现线程hang住的情况，就是在于 releaseShared 有竞争的情况下，可能会有队列中处于等待状态的节点因为第一个线程完成释放唤醒，第二个线程获取到锁，但还没设置好head，又有新线程释放锁，但是读到老的 head 状态为0导致释放但不唤醒，最终后一个等待线程既没有被释放线程唤醒，也没有被持锁线程唤醒





---





## CountDownLatch

countDownLatch 是一个同步工具类，它允许一个或多个线程等待直到其他线程执行完再执行。

### 如何使用？

#### 1. 构造方法

```java
// count 指定需要等待的线程数量
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

#### 2. 常用方法

```java
// 调用此方法的线程会被阻塞，直到 CountDownLatch 的 count 为 0
public void await() throws InterruptedException 

// 和上面的 await() 作用基本一致，只是可以设置一个最长等待时间
public boolean await(long timeout, TimeUnit unit) throws InterruptedException

// 会将 count 减 1，直至为 0
public void countDown()
```

#### 3. 基本使用

场景：假设有小朱、小胡、小陈、小陈男要出去玩，每个人抽象为一条线程，要等到人齐后在出发，为主线程，那代码如下：

```java
public class CountDownLatchDemo1 {

    // 线程类
     static class Person implements Runnable{
        String name;
        CountDownLatch countDownLatch;
        static Random random = new Random();
        public Person(String name, CountDownLatch countDownLatch){
            this.name = name;
            this.countDownLatch = countDownLatch;
        }
        @Override
        public void run() {
            int time = random.nextInt(500);
            try {
                TimeUnit.MILLISECONDS.sleep(time);
                System.out.printf("%s 花了%ds到达\n", name, time);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }finally {
                // 执行完任务 countDown
                countDownLatch.countDown();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException{
        CountDownLatchDemo1 demo = new CountDownLatchDemo1();
        CountDownLatch countDownLatch = new CountDownLatch(4);

        Thread t1 = new Thread(new Person("小朱", countDownLatch));
        Thread t2 = new Thread(new Person("小胡", countDownLatch));
        Thread t3 = new Thread(new Person("小陈", countDownLatch));
        Thread t4 = new Thread(new Person("小陈男", countDownLatch));
        t1.start();
        t2.start();
        t3.start();
        t4.start();

        System.out.println("等待中...");
        // 主线程等待
        countDownLatch.await();
        System.out.println("人齐，冲冲冲");
    }

}
```

结果

```
等待中...
小朱 花了302s到达
小陈男 花了320s到达
小胡 花了377s到达
小陈 花了432s到达
人齐，冲冲冲
```

#### 4. CountDownLatch 和 join

当主函数做出下面的改动

```java
        System.out.println("等待中...");
        // 主线程等待
//        countDownLatch.await();
        t1.join();
        t2.join();
        t3.join();
        t4.join();
        System.out.println("人齐，冲冲冲");
```

同样可以达到目的。那为什么要用 CountDownLatch？

**假设我们为了减少创建线程的开销，上面线程执行完毕后会被回收到线程池，也就是没有死亡，那就没办法通过 join 判断线程是否执行完毕，这时 CountDownLatch 就有明显的优势了。**



![](img/latch.png)

### 源码分析

构造函数

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

private static final class Sync extends AbstractQueuedSynchronizer {
    Sync(int count) {
        // 直接将 state 设置为 count
        setState(count);
    }
    ...
}
```

await 方法

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}


public final void acquireSharedInterruptibly(int arg) //arg == 1
        throws InterruptedException {
    // 如果线程中断，抛给上层并重置中断标志
    if (Thread.interrupted())
        throw new InterruptedException();
    // 小于0说明state不为0
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}

protected int tryAcquireShared(int acquires) {
    // state 为0返回1，否则返回-1
    return (getState() == 0) ? 1 : -1;
}

// state不为0执行doAcquireSharedInterruptibly，其实就是获取共享锁
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    // 添加到同步队列
    final Node node = addWaiter(Node.SHARED);
    try {
        for (;;) {
            final Node p = node.predecessor();
            // 如果 p 为头结点
            if (p == head) {
                // 只要 state 不等于 0，那么这个方法返回 -1
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 唤醒阻塞的线程并设置头结点为node
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    return;
                }
            }
            // 假设还没有 countDown 到0，则 r == -1,会进来这个分支
            // shouldParkAfterFailedAcquire 会阻塞线程
            // 也就是调用 await 的线程被阻塞在这里
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } catch (Throwable t) {
        cancelAcquire(node);
        throw t;
    }
}
```

countDown 方法

```java
public void countDown() {
    sync.releaseShared(1);
}

public final boolean releaseShared(int arg) {
    // 每次 try 将 state 减 1，如果 state 为0，tryReleaseShared 返回true
    if (tryReleaseShared(arg)) {
        // 唤醒 await 的线程
        doReleaseShared();
        return true;
    }
    return false;
}

// 每次将 state 减1， 如果 state 为0，返回 true
protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

也就是 CountDown 方法实际上就是将 state 减 1，直至 state 为 0，开始唤醒等待的线程

假设执行 CountDown 后 state 为 0，则执行 doReleaseShared 开始唤醒等待的线程

```Java
private void doReleaseShared() {
    /*
     * 以下的循环做的事情就是，在队列存在后继线程的情况下，唤醒后继线程；
     * 或者由于多线程同时释放共享锁由于处在中间过程，读到head节点等待状态为0的情况下，
     * 虽然不能unparkSuccessor，但为了保证唤醒能够正确稳固传递下去，设置节点状态为PROPAGATE。
     * 这样的话获取锁的线程在执行setHeadAndPropagate时可以读到PROPAGATE，从而由获取锁的线程去释放后继等待线程。
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                // 唤醒阻塞队列线程
                unparkSuccessor(h);
            }
            // 如果h节点的状态为0，需要设置为PROPAGATE用以保证唤醒的传播。
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // todo
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

假设线程被唤醒，它会从 doAcquireSharedInterruptibly 中阻塞的地方继续执行

```java
private void doAcquireSharedInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r); // 2. 这里是下一步
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                // 1. 唤醒后这个方法返回
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

假设此时没有中断，于是被唤醒的线程 A 执行 setHeadAndPropagate（之所以执行是现在 state 已经为0）

```java
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    // 线程A设置自己为头结点
    setHead(node);
	// 线程A唤醒后会继续尝试唤醒 node 后面的节点
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            // 此时头结点已经被设置为原来线程A所在的节点了
            doReleaseShared();
    }
}
```

此时又回到 doReleaseShared

```java
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        // head为null，阻塞队列为空
        // head为tail，节点都已经被唤醒
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     // 这里失败是因为有节点入队将 ws 设置为-1
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE)) // todo
                continue;                // loop on failed CAS
        }
        // 如果 h == head，说明刚刚被唤醒的线程还没来得及将自己设置为头结点
        // h != head，则刚唤醒的线程成功设置自己为头结点，继续上面的for循环，唤醒后继节点
        if (h == head)                   // loop if head changed
            break;
    }
}
```















Reference：

[AbstractQueuedSynchronizer源码解读](https://www.cnblogs.com/micrari/p/6937995.html)

[一行一行源码分析清楚 AbstractQueuedSynchronizer](https://javadoop.com/post/AbstractQueuedSynchronizer-3)



