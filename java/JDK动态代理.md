## 代理模式

代理模式：给某一个对象提供一个代理，并由代理来控制对真实对象的访问。

代理模式角色主要分为三种：

+ Subject（抽象主题角色）：定义代理类和真实主题的公共对外方法，也是代理类代理真实主题的方法
+ RealSubject（真实主题角色）：真正实现业务逻辑的类
+ Proxy（代理主题角色）：用来代理和封装真实主题



<img src="img/400px-Proxy_pattern_diagram.svg.png" style="zoom:150%;" />



如果按照代理模式使用目的，常见的代理模式有：

- 远程(Remote)代理：为一个位于不同的地址空间的对象提供一个本地 的代理对象，这个不同的地址空间可以是在同一台主机中，也可是在 另一台主机中，远程代理又叫做大使(Ambassador)。

  

- 虚拟(Virtual)代理：如果需要创建一个资源消耗较大的对象，先创建一个消耗相对较小的对象来表示，真实对象只在需要时才会被真正创建。

  

- Copy-on-Write代理：它是虚拟代理的一种，把复制（克隆）操作延迟 到只有在客户端真正需要时才执行。一般来说，对象的深克隆是一个 开销较大的操作，Copy-on-Write代理可以让这个操作延迟，只有对象被用到的时候才被克隆。

  

- 保护(Protect or Access)代理：控制对一个对象的访问，可以给不同的用户提供不同级别的使用权限。

  

- 缓冲(Cache)代理：为某一个目标操作的结果提供临时的存储空间，以便多个客户端可以共享这些结果。

  

- 防火墙(Firewall)代理：保护目标不让恶意用户接近。



在 `Java` 中如果按照字节码创建可以分为静态代理和动态代理。



## 静态代理

所谓**静态**也就是在**程序运行前**就已经存在代理类的**字节码文件**，代理类和真实主题角色的关系在运行前就确定了。

使用静态代理时，需要定义接口或者父类（Subject），被代理对象（RealSubject）和代理对象（Proxy）一起实现相同接口或者继承相同父类。

### 例子

定义抽象主题角色（Subject）：

```java
public interface HelloInterface {
    void sayHello();
}
```

定义实现接口的真实对象（RealSubject）

```Java
public class RealHello implements HelloInterface{
    @Override
    public void sayHello() {
        System.out.println("hello");
    }
}
```

定义代理对象，实现和真实对象一样接口的代理类

```Java
public class HelloProxy implements HelloInterface {

    // 保存真实对象
    private RealHello hello;

    public HelloProxy(RealHello hello){
        this.hello = hello;
    }

    @Override
    public void sayHello() {
        System.out.println("before sayHello invoke");
        // 调用真实对象的方法
        hello.sayHello();
        System.out.println("after sayHello invoke");
    }
}
```

客户端测试

```Java
public class Client {
    public static void main(String[] args) {
        RealHello realHello = new RealHello();
        HelloProxy proxy = new HelloProxy(realHello);
        proxy.sayHello();
    }
}
```

结果：

```
before sayHello invoke
hello
after sayHello invoke
```



### 静态代理总结：

优点：可以在不修改目标对象功能前提下，对目标对象的功能进行扩展。

缺点：

一、代理对象要实现与目标对象相同的接口，有两种方式：

+ 只维护一个代理类，由这个代理类实现每一个与目标对象一致的接口，这样会导致代理类混杂，庞大。
+ 对每一个目标对象实现代理类，这样会生成过多代理类。

二、当接口增加、删除、修改方法时，目标对象和代理对象都需要修改，不易维护。



## 动态代理

代理类在运行时动态生成。下面以 JDK 动态代理为例。

定义抽象主题角色（Subject）：

```Java
interface Say{
    void sayHello();
    void sayBye();
}
```

定义实现接口的真实对象（RealSubject）：

```java
class Hello implements Say{
    @Override
    public void sayHello() {
        System.out.println("hello");
    }

    @Override
    public void sayBye() {
        System.out.println("bye");
    }
    // Hello 类自己的方法
    public void helloMethod(){
        System.out.println("...");
    }
}
```



通过实现 `InnovationHandle`自定义 `Handle`，用来实现对代理目标的扩展。

> InnovationHandle：代理实例调用处理程序实现的接口。
>
> 每个代理实例都有一个关联的 InnovationHandle，当在代理实例上调用方法时，该方法会被分派到 invoke 中执行，我们也是在 invoke 中对方法进行扩展



```Java
class HelloHandle implements InvocationHandler{
    private Object obj;

    public HelloHandle(Object obj) {
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable
    {
        // 对 sayHello 方法进行扩展
        if (method.getName().equals("sayHello")){
            System.out.println("before");
            method.invoke(obj, args);
            System.out.println("after");
        }
        // 其他方法直接执行
        else
            method.invoke(obj, args);
        return null;
    }
}
```

调用代理对象：

```Java
public class ProxyTest {

    public static void main(String[] args) throws Throwable {
        System.setProperty("jdk.proxy.ProxyGenerator.saveGeneratedFiles", "true");
        // 得到目标对象实现的接口数组
        Class[] ints = Hello.class.getInterfaces();
        // 得到目标类的类加载器
        ClassLoader cl = Hello.class.getClassLoader();
        // 代理类关联的 Handle
        HelloHandle helloHandle = new HelloHandle(new Hello());
        // 生成代理类实例
        Say hello = (Say) Proxy.newProxyInstance(cl, ints, helloHandle);
        hello.sayBye();
        System.out.println("----------------");
        hello.sayHello();
    }
}
```

结果：

```
bye
----------------
before
hello
after
```



### 动态代理总结

优点：解决了静态代理中冗余代理类问题。

缺点：JDK 动态代理是基于接口设计实现的，如果没有接口，会抛异常



## JDK动态代理源码

**以下代码都是基于 JDK11**

因为我们是通过 `newProxyInstance` 生成的，所以以 `newProxyInstance` 作为切入点 

```Java
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h) {
    Objects.requireNonNull(h);

    final Class<?> caller = System.getSecurityManager() == null
        ? null
        : Reflection.getCallerClass();

    // 查找或生成指定的代理类的构造函数
    Constructor<?> cons = getProxyConstructor(caller, loader, interfaces);

    // 根据构造函数生成代理实例
    return newProxyInstance(caller, cons, h);
}
```

getProxyConstructor 方法

```java
private static Constructor<?> getProxyConstructor(Class<?> caller,
                                                  ClassLoader loader,
                                                  Class<?>... interfaces)
{
    // optimization for single interface
    // 优化只有一个接口的情况
    if (interfaces.length == 1) {
        Class<?> intf = interfaces[0];
        if (caller != null) {
            checkProxyAccess(caller, loader, intf);
        }
        // 返回代理类构造函数，也就是说代理类的生成也在这里实现
        return proxyCache.sub(intf).computeIfAbsent(
            loader,
            // 匿名函数生成代理类构造函数
            (ld, clv) -> new ProxyBuilder(ld, clv.key()).build()
        );
    } else {
        // interfaces cloned
        final Class<?>[] intfsArray = interfaces.clone();
        if (caller != null) {
            checkProxyAccess(caller, loader, intfsArray);
        }
        final List<Class<?>> intfs = Arrays.asList(intfsArray);
        return proxyCache.sub(intfs).computeIfAbsent(
            loader,
            (ld, clv) -> new ProxyBuilder(ld, clv.key()).build()
        );
    }
}
```

这里注意 ProxyBuilder 构造函数中有对代理类实现接口的数量的限制

```Java
ProxyBuilder(ClassLoader loader, List<Class<?>> interfaces) {
    if (!VM.isModuleSystemInited()) {
        throw new InternalError("Proxy is not supported until "
                                + "module system is fully initialized");
    }
    // 接口数量不能大于65535
    if (interfaces.size() > 65535) {
        throw new IllegalArgumentException("interface limit exceeded: "
                                           + interfaces.size());
        //...
}
```



build 方法

```Java
Constructor<?> build() {
    // 生成代理类的 Class 对象
    Class<?> proxyClass = defineProxyClass(module, interfaces);
    final Constructor<?> cons;
    try {
        // 通过代理类的class对象获得构造函数
        // 这个构造函数的参数为 InvocationHandler.class 
        cons = proxyClass.getConstructor(constructorParams);
    } catch (NoSuchMethodException e) {
        throw new InternalError(e.toString(), e);
    }
    AccessController.doPrivileged(new PrivilegedAction<Void>() {
        public Void run() {
            // 设置构造函数可访问
            cons.setAccessible(true);
            return null;
        }
    });
    return cons;
}
```



生产 Class 对象的 defineProxyClass 方法

```Java
private static Class<?> defineProxyClass(Module m, List<Class<?>> interfaces) {
    String proxyPkg = null;     // package to define proxy class in
    int accessFlags = Modifier.PUBLIC | Modifier.FINAL;

    // 下面的for循环是寻找非public接口
    // 标识符非public的接口应该处于一个包中，否则报错
    for (Class<?> intf : interfaces) {
        int flags = intf.getModifiers();
        if (!Modifier.isPublic(flags)) {
            accessFlags = Modifier.FINAL;  // non-public, final
            String pkg = intf.getPackageName();
            if (proxyPkg == null) {
                proxyPkg = pkg;
            } else if (!pkg.equals(proxyPkg)) {
                throw new IllegalArgumentException(
                    "non-public interfaces from different packages");
            }
        }
    }

    // 确定包名，如果没有非Public代理接口，包名使用com.sun.proxy
    if (proxyPkg == null) {
        // all proxy interfaces are public
        proxyPkg = m.isNamed() ? PROXY_PACKAGE_PREFIX + "." + m.getName()
            : PROXY_PACKAGE_PREFIX;
    } else if (proxyPkg.isEmpty() && m.isNamed()) {
        throw new IllegalArgumentException(
            "Unnamed package cannot be added to " + m);
    }

    if (m.isNamed()) {
        if (!m.getDescriptor().packages().contains(proxyPkg)) {
            throw new InternalError(proxyPkg + " not exist in " + m.getName());
        }
    }

    
    long num = nextUniqueNumber.getAndIncrement();
    // 为代理类取名
    // proxyClassNamePrefix = "$Proxy"
    String proxyName = proxyPkg.isEmpty()
        ? proxyClassNamePrefix + num
        : proxyPkg + "." + proxyClassNamePrefix + num;
	
    ClassLoader loader = getLoader(m);
    trace(proxyName, m, loader, interfaces);

    // 综合上面可知，对于第一个生成的代理类，如果全部接口都是Public
    // 则生成的类为：com.sun.proxy.$Proxy0.class

    // 生成代理类字节码
    byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
        proxyName, interfaces.toArray(EMPTY_CLASS_ARRAY), accessFlags);
    try {
        Class<?> pc = UNSAFE.defineClass(proxyName, proxyClassFile,
                                         0, proxyClassFile.length,
                                         loader, null);
        reverseProxyCache.sub(pc).putIfAbsent(loader, Boolean.TRUE);
        return pc;
    } catch (ClassFormatError e) {
        throw new IllegalArgumentException(e.toString());
    }
}
```



生成代理字节码的方法 generateProxyClass

```Java
static byte[] generateProxyClass(final String name,
                                 Class<?>[] interfaces,
                                 int accessFlags)
{
    ProxyGenerator gen = new ProxyGenerator(name, interfaces, accessFlags);
    // 通过ProxyGenerator生成字节数组
    // 这个方法直接操纵生成代理类字节码
    final byte[] classFile = gen.generateClassFile();
	// 以下是保存代理类
    // saveGeneratedFiles相当于开关，是否选择保存生成的代理类
    if (saveGeneratedFiles) {
        java.security.AccessController.doPrivileged(
            new java.security.PrivilegedAction<Void>() {
                public Void run() {
                    try {
                        int i = name.lastIndexOf('.');
                        Path path;
                        if (i > 0) {
                            Path dir = Path.of(name.substring(0, i).replace('.', File.separatorChar));
                            Files.createDirectories(dir);
                            path = dir.resolve(name.substring(i+1, name.length()) + ".class");
                        } else {
                            path = Path.of(name + ".class");
                        }
                        Files.write(path, classFile);
                        return null;
                    } catch (IOException e) {
                        throw new InternalError(
                            "I/O exception saving generated file: " + e);
                    }
                }
            });
    }

    return classFile;
}
```

是否保存代理类的开关定义：saveGeneratedFiles

```Java
/** debugging flag for saving generated class files */
private static final boolean saveGeneratedFiles =
    java.security.AccessController.doPrivileged(
    new GetBooleanAction(
        "jdk.proxy.ProxyGenerator.saveGeneratedFiles")).booleanValue();
```

这也是为什么我们执行 `System.setProperty("jdk.proxy.ProxyGenerator.saveGeneratedFiles", "true");` 会生成代理类 Class 文件的原因。



## 生成的代理类

```Java
final class $Proxy0 extends Proxy implements Say {
    private static Method m1;
    private static Method m4;
    private static Method m2;
    private static Method m3;
    private static Method m0;

    // 构造函数，传入的是我们实现的Handle
    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHello() throws  {
        try {
            super.h.invoke(this, m4, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final void sayBye() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m4 = Class.forName("Say").getMethod("sayHello");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("Say").getMethod("sayBye");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

1. 从生成的代码可以看出，生成的代理类就是一个实现了我们传入接口的实现类，属于被代理类中的方法并不会在代理类中出现。

2. 代理类继承于 Proxy



#### 构造函数

```Java
public $Proxy0(InvocationHandler var1) throws  {
    super(var1);
}
```

在代理类中，构造函数参数是 `InvocationHandler`，然后调用了父类 `Proxy` 的构造函数，这里其实就是为该代理实例关联我们传入的 `InvocationHandler`，当我们调用代理类的方法时，该方法会被分派到`InvocationHandler.invoke` 方法中。



#### 代理方法

下面以 `sayHello` 为例：

首先在代理类中的静态代码块初始化了 `sayHello` 的方法，当我们调用了代理类的 `sayHello` 方法，这个方法通过我们在构造函数传入的 `Handle` 执行了 `invoke` 方法，在 `invoke` 方法中，我们执行了对被代理类 `sayHello` 方法的调用。



#### 其他方法

通过生成的代理类发现，不止接口中的方法，`hashcode()`, `equals()` and `toString()` 方法也被分派到 `InvocationHandle` 中。

之所以这样设计是因为代理类用于间接访问目标对象，就客户端代码而言，应该隐藏代理的存在。

我们以 toString 为例

```Java
public final String toString() throws  {
    try {
        return (String)super.h.invoke(this, m2, (Object[])null);
    } catch (RuntimeException | Error var2) {
        throw var2;
    } catch (Throwable var3) {
        throw new UndeclaredThrowableException(var3);
    }
}
```

toString 方法的返回值是 `invoke` 的返回值，所以，为了返回被代理对象的 toString 方法，我们可以将上面的 invoke 进行修改

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
	// 保存函数执行返回的值
    Object returnMsg;
    if (method.getName().equals("sayHello")){
        System.out.println("before");
        returnMsg = method.invoke(obj, args);
        System.out.println("after");
    }
    else
        returnMsg = method.invoke(obj, args);
    // 返回执行结果
    return returnMsg;
}
```

如果我们不将结果返回，直接返回 NULL，会导致上面三个函数的返回值都为NULL。

因为 `equals` 方法的重写，会发生一些有趣的现象：

```Java
public static void main(String[] args) throws Throwable {
    System.setProperty("jdk.proxy.ProxyGenerator.saveGeneratedFiles", "true");
    // 得到目标对象实现的接口数组
    Class[] ints = Hello.class.getInterfaces();
    // 得到目标类的类加载器
    ClassLoader cl = Hello.class.getClassLoader();
    Hello h = new Hello();
    // 代理类关联的 Handle
    HelloHandle helloHandle = new HelloHandle(h);
    Say hello = (Say) Proxy.newProxyInstance(cl, ints, helloHandle);
    // h：被代理对象
    // hello：代理对象
    System.out.println(hello.equals(h));
    System.out.println(h.equals(hello));
}
```

结果为：

```
true
false
```

因为我们没有重写任何 equals，所以上面两个语句调用的都是默认的 equals 方法，如下：

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

因为默认 equals 方法比较的是对象内存中的地址，所以按道理两个应该都为 false，那为什么出现 true？

原因就是，对于代理对象 `hello` 来说 ，它调用的 equals 方法被分派到 `InvocationHandle` 中，由代理对象执行，也就是说 `hello.equals(h)` 相当于 `h.equals(h)`，所以返回 true。



## 相关问题

#### 一、为什么代理类接口数不能超过 65535？

因为 Java 字节码中，用 u2 （也就是两个字节）来存储接口数，因此也就决定了接口数不能超过65535，否则会编译出错。

// TODO













### Reference：

[图说设计模式](https://design-patterns.readthedocs.io/zh_CN/latest/structural_patterns/proxy.html)

[https://laijianfeng.org/2018/12/Java-%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86%E8%AF%A6%E8%A7%A3/](https://laijianfeng.org/2018/12/Java-动态代理详解/)