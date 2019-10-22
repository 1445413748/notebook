### Future 接口

`Future`接口代表异步计算的结果，通过`Future`接口提供的方法可以查看异步计算是否执行完成，或者等待执行结果并获取执行结果，同时还可以取消执行。

```java
public interface Future<V> {
    // 尝试取消正在执行的线程，如果任务执行完成，则cancel失败，如果任务还没有开始，则不会被执行
    // 如果任务正在执行，参数指明任务应该执行完成还是被中断
    boolean cancel(boolean mayInterruptIfRunning);
    
    // 任务是否在执行完成前被取消
    boolean isCancelled();
     
    // 任务是否执行完成
    // 任务执行过程中被取消或者发生异常也是执行成功
    boolean isDone();
    
    // 得到任务执行完的结果,如果任务没有执行完成则阻塞等待
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}

```

### FutureTask 类

```
Future      Runnable
   \           /
    \         /
   RunnableFuture
          |
          |
      FutureTask
      
FutureTask 实现了 RunnableFuture 接口，而 RunnableFuture 继承自 Future、Runnable，所以 FutureTask 间接实现了 Future、Runnable 接口。
```

#### 使用

```java
public class FutureTest {

    static class task implements Callable<Integer>{
        @Override
        public Integer call() throws Exception {
            TimeUnit.SECONDS.sleep(1);
            return 3;
        }
    }

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        FutureTask<Integer> futureTask = new FutureTask<>(new task());
        Thread thread = new Thread(futureTask);
        // 执行
        thread.start();
        if (!futureTask.isDone()){
            System.out.println("thread is not done");
            TimeUnit.SECONDS.sleep(1);
        }

        // 获取值，如果线程还没有执行完成则阻塞
        int res = futureTask.get();
        System.out.println("complete! result is " + res);
    }

}
```



#### 主要成员

```java
/** The underlying callable; nulled out after running */
// 被提交的任务
private Callable<V> callable;

/** The result to return or exception to throw from get() */
// 任务执行完返回的结果或者异常信息
private Object outcome; // non-volatile, protected by state reads/writes

/** The thread running the callable; CASed during run() */
// 执行任务的线程
private volatile Thread runner;

/** Treiber stack of waiting threads */
// 等待节点
private volatile WaitNode waiters;

// 表示当前 Task 状态
private volatile int state;
```

#### 七种状态

```Java
// 初始态
// 新任务，还没有被执行
private static final int NEW          = 0;

// 中间状态
// 任务已经完成或者执行任务过程发生异常
// 此时结果或者异常信息还没有保存到 outcome 中
private static final int COMPLETING   = 1;

// 最终态
// 任务执行完成并且结果已经保存到 outcome 中
private static final int NORMAL       = 2;

// 最终态
// 任务执行发生异常且异常信息已经保存到 outcome 中
private static final int EXCEPTIONAL  = 3;

// 最终态
// 任务还没开始执行或者已经开始执行但是还没有执行完成的时候
// 用户调用 cancel(false)取消任务但没中断线程
private static final int CANCELLED    = 4;

// 中间状态
// 任务还没开始执行或者已经开始执行但是还没有执行完成的时候
// 取消任务并且要中断任务执行线程但是还没有中断任务执行线程
private static final int INTERRUPTING = 5;

// 最终态
// 调用 interrupt 中断线程
private static final int INTERRUPTED  = 6;
```

几种状态可能的转化过程

```
NEW -> COMPLETING -> NORMAL
NEW -> COMPLETING -> EXCEPTIONAL
NEW -> CANCELLED
NEW -> INTERRUPTING -> INTERRUPTED
```

#### 构造函数

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    // 设置状态
    this.state = NEW;       // ensure visibility of callable
}

public FutureTask(Runnable runnable, V result) {
    // 将 runnable 封装成 Callable 对象
    this.callable = Executors.callable(runnable, result);
    this.state = NEW;       // ensure visibility of callable
}
```

#### run 方法

```java
public void run() {
    // 如果状态不是NEW，说明任务执行过
    // 如果状态是NEW，将当前执行线程保存在runner中
    // CAS操作失败说明多个线程抢夺任务，当前线程抢夺失败
    if (state != NEW ||
        !RUNNER.compareAndSet(this, null, Thread.currentThread()))
        return;
    try {
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
                // 执行任务
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                // 设置异常
                setException(ex);
            }
            if (ran)
                // 设置结果
                set(result);
        }
    } finally {
        // runner must be non-null until state is settled to
        // prevent concurrent calls to run()
        runner = null;
        // state must be re-read after nulling runner to prevent
        // leaked interrupts
        int s = state;
        // 任务被中断，执行中断
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}


/* setException */
protected void setException(Throwable t) {
    // 设置状态 NEW -> COMPLETING
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        // 保存异常信息
        outcome = t;
        // 设置状态为 EXCEPTIONAL
        STATE.setRelease(this, EXCEPTIONAL); // final state
        finishCompletion();
    }
}

/* set */
protected void set(V v) {
    if (STATE.compareAndSet(this, NEW, COMPLETING)) {
        // 这里保存的是返回结果
        outcome = v;
        STATE.setRelease(this, NORMAL); // final state
        finishCompletion();
    }
}
```

#### get 方法

返回任务执行完成结果，如果任务还没有执行完成，则调用方会阻塞到任务执行结束返回结果为止。

```java
// 这个是没有超时等待的get
public V get() throws InterruptedException, ExecutionException {
    // 获取任务状态
    int s = state;
    // 还没有完成，阻塞
    if (s <= COMPLETING)
        s = awaitDone(false, 0L);
    // 调用report返回最终结果
    return report(s);
}
```

阻塞等待的方法 awaitDone

```java
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    long startTime = 0L;    // Special value 0L means not yet parked
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
        int s = state;
        // 判断任务是否已经结束，这里的结束有：正常结束，异常结束，中断
        // 置空Thread，返回
        if (s > COMPLETING) {
            if (q != null)
                q.thread = null;
            return s;
        }
        // 如果处于中间状态COMPLETING说明任务结束，还没有给outcome赋值
        // 让出线程执行权就好了，没必要阻塞线程
        else if (s == COMPLETING)
            // We may have already promised (via isDone) that we are done
            // so never return empty-handed or throw InterruptedException
            Thread.yield();
        // 如果线程被中断，从等待队列中移除
        // 可能有多个线程调用了 get 方法，所以有等待队列
        else if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }
        // 等待节点空，构造节点
        else if (q == null) {
            if (timed && nanos <= 0L)
                return s;
            // 当前线程被包装成WaitNode
            q = new WaitNode();
        }
        // 如果节点还没有进入队列则入队
        else if (!queued)
            queued = WAITERS.weakCompareAndSet(this, q.next = waiters, q);
        // 如果设置等待时间，则计算
        else if (timed) {
            final long parkNanos;
            if (startTime == 0L) { // first time
                startTime = System.nanoTime();
                if (startTime == 0L)
                    startTime = 1L;
                parkNanos = nanos;
            } else {
                long elapsed = System.nanoTime() - startTime;
                // 超出等待时间
                if (elapsed >= nanos) {
                    removeWaiter(q);
                    return state;
                }
                parkNanos = nanos - elapsed;
            }
            // nanoTime may be slow; recheck before parking
            if (state < COMPLETING)
                LockSupport.parkNanos(this, parkNanos);
        }
        else
            LockSupport.park(this);
    }
}
```

**从上面可以知道，线程调用 get 方法但没有立即返回不一定是被阻塞，有可能是任务处于 COMPLETING 状态，线程让出了执行权而已。**

#### 获取结果 report

```java
private V report(int s) throws ExecutionException {
    // 如果任务是正常执行完，outcome 保存的是返回结果
    // 如果任务抛出异常，outcome 保存的是对应的 exception
    Object x = outcome;
    // 正常执行完
    if (s == NORMAL)
        return (V)x;
    if (s >= CANCELLED)
        throw new CancellationException();
    // 任务执行异常，抛出异常
    throw new ExecutionException((Throwable)x);
}
```



#### 取消任务 cancel

```java
public boolean cancel(boolean mayInterruptIfRunning) {
    // 任务已经结束
    // 根据mayInterruptIfRunning设置节点状态失败
    // 都返回 false
    if (!(state == NEW && STATE.compareAndSet
          (this, NEW, mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        // 如果需要中断线程
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    // 中断线程
                    t.interrupt();
            } finally { // final state
                // 中断后设置状态
                STATE.setRelease(this, INTERRUPTED);
            }
        }
    } finally {
        // 唤醒阻塞线程
        finishCompletion();
    }
    return true;
}
```

#### 唤醒线程finishCompletion

```java
private void finishCompletion() {
    // assert state > COMPLETING;
    for (WaitNode q; (q = waiters) != null;) {
        if (WAITERS.weakCompareAndSet(this, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }

    done();

    callable = null;        // to reduce footprint
}
```

这个方法在 执行完 run 和 cancel 方法后会被调用，其作用就是遍历 waiters 链表，唤醒被阻塞的线程，也就是调用 get 方法被阻塞的线程，在这个方法中有个 done 方法，可以给调用者重写这个方法，执行想要的功能。

#### 判断任务状态

```Java
public boolean isCancelled() {
    return state >= CANCELLED;
}

public boolean isDone() {
    return state != NEW;
}
```

