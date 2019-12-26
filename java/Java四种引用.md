### 强引用

特点：我们平常典型编码Object obj = new Object()中的obj就是强引用。通过关键字new创建的对象所关联的引用就是强引用。 当JVM内存空间不足，JVM宁愿抛出OutOfMemoryError运行时错误（OOM），使程序异常终止，也不会靠随意回收具有强引用的“存活”对象来解决内存不足的问题。对于一个普通的对象，如果没有其他的引用关系，只要超过了引用的作用域或者显式地将相应（强）引用赋值为 null，就是可以被垃圾收集的了，具体回收时机还是要看垃圾收集策略。

### 软引用

特点：软引用通过SoftReference类实现。 软引用的生命周期比强引用短一些。**只有当 JVM 认为内存不足时，才会去试图回收软引用指向的对象**：即JVM 会确保在抛出 `OutOfMemoryError`之前，清理软引用指向的对象。

软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收器回收，Java虚拟机就会把这个软引用加入到与之关联的引用队列中。后续，我们可以调用ReferenceQueue的poll()方法来检查是否有它所关心的对象被回收。如果队列为空，将返回一个null,否则该方法返回队列中前面的一`Reference`对象。

```java
// 引用队列
ReferenceQueue<String> referenceQueue = new ReferenceQueue<>();
String str = new String("abc");
// 将对象包装成软引用对象
SoftReference<String> softReference = new SoftReference<>(str, referenceQueue);
```

如果内存不足，`JVM` 会将软引用中的对象置为 `null`，然后通知垃圾回收器进行回收，大致过程如下：

```java
if (JVM 内存不足){
    // 将软引用中对象置为null
    str = null;
    // 通知垃圾回收器回收
    System.gc();
}
```



应用场景：软引用通常用来实现内存敏感的缓存。如果还有空闲内存，就可以暂时保留缓存，当内存不足时清理掉，这样就保证了使用缓存的同时，不会耗尽内存。

### 弱引用

弱引用通过 `WeakReference` 类实现。 弱引用的生命周期比软引用短。在垃圾回收器线程扫描它所管辖的内存区域的过程中，**一旦发现了具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存**。

```java
String str = new String("abc");
// 将str包装成弱引用对象
WeakReference<String> weakReference = new WeakReference<>(str);
```

当 `JVM` 发现弱引用对象，会将弱引用中的对象置为 `null`，然后通过垃圾回收器回收

```java
if (JVM 发现弱引用对象){
    // 设置弱引用中对象为null
    str = null;
    // 通知垃圾回收器回收
    System.gc();
}
```



**由于垃圾回收器是一个优先级很低的线程，因此不一定会很快回收弱引用的对象。**弱引用可以和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java虚拟机就会把这个弱引用加入到与之关联的引用队列中。

```java
// 目标类
class Target{
    int i;
    byte[] buffer = new byte[1024];
    public Target(int i){
        this.i = i;
    }

    @Override
    protected void finalize() throws Throwable {
        System.out.println("Target " + i + " in finalized");
    }
}

// 虚引用包装类
class WeakTarget extends WeakReference<Target> {
    int i;
    public WeakTarget(Target referent, ReferenceQueue<? super Target> q) {
        super(referent, q);
        this.i = referent.i;
    }

    @Override
    protected void finalize() throws Throwable {
        System.out.println("weakTarget" + i + " in finalized");
    }

    @Override
    public String toString() {
        return "WeakTarget wrapper target " + i;
    }
}


public class WeakReferenceTest {

    public static void main(String[] args) throws InterruptedException {
        // 引用队列
        ReferenceQueue<Target> queue = new ReferenceQueue<>();
        WeakTarget weakTarget = new WeakTarget(new Target(1), queue);
        // 调用垃圾回收
        System.gc();
        // 等待垃圾回收器回收
        Thread.sleep(1000);
        Reference<? extends Target> reference;
        // 如果回收对象target，则会将包装对象的弱引用加入引用队列
        if ((reference = queue.poll()) != null)
            System.out.println(reference);
        else
            System.out.println("queue empty");
    }
}
```

结果：

```
Target 1 in finalized
WeakTarget wrapper target 1
```

如果注释掉

```java
// 调用垃圾回收
System.gc();
// 等待垃圾回收器回收
Thread.sleep(1000);
```

则结果为

```
queue empty
```

原因就是没有进行垃圾回收，`JVM` 没有发现弱引用对象 `Target(1)`，自然不会回收它并将相应的弱引用加入关联的引用队列。



应用场景：弱应用同样可用于内存敏感的缓存。

### 虚引用

特点：虚引用也叫幻象引用，通过`PhantomReference`类来实现。**无法通过虚引用访问对象的任何属性或函数**。幻象引用仅仅是提供了一种确保对象被 fnalize 以后，做某些事情的机制。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收器回收。虚引用必须和引用队列 （ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。

```java
ReferenceQueue queue = new ReferenceQueue ();
PhantomReference pr = new PhantomReference (object, queue); 
```

程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取一些程序行动。

应用场景：可用来跟踪对象被垃圾回收器回收的活动，当一个虚引用关联的对象被垃圾收集器回收之前会收到一条系统通知。



### Reference

[理解Java的强引用、软引用、弱引用和虚引用](https://juejin.im/post/5b82c02df265da436152f5ad)

