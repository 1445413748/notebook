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

假设我们为了减少创建线程的开销，上面线程执行完毕后会被回收到线程池，也就是没有死亡，那就没办法通过 join 判断线程是否执行完毕，这时 CountDownLatch 就有明显的优势了。



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

count 方法

