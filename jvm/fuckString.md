### Java中的常量池



#### 全局字符串池（string literal pool）

在 Hotspot 中，实现 string pool 的是一个 StringTable 类，它的本质是HashSet\<String> 。这是一个纯运行时结构，而且是惰性（lazy）维护的。==它只存储 java.lang.String 实例的引用，而不存储 String 对象的内容。==

#### Class 文件中的常量池（class constant pool）

java文件被编译成class文件之后，也就是会生成class常量池，class常量池主要存放各种字面量（Literal）和符号引用（Symbolic Reference）

字面量： 文本字符串、被声明为 final 的常量值等。

符号引用是一组符号用来描述所引用的目标，符号可以是任何形式的字面量，主要包括下面三类常量：

+ 类和接口的全限定名
+ 字段的名称和描述符
+ 方法的名称和描述符

[JVM里的符号引用如何存储？ - RednaxelaFX的回答 - 知乎](https://www.zhihu.com/question/30300585/answer/51335493)

#### 运行时常量池（runtime constant pool）

当主动使用某个类时，必须经过加载、连接、初始化，其中连接还包含了验证、准备、解析阶段。当加载完某个类后，JVM 会将 class 常量池中的内容存放到运行时常量池中。由此可以得到运行时常量池也是每个类都有一个。

==注意，JVM规范里明确指定resolve阶段可以是lazy的。==

### 字符串在三个池中的苟且

首先看一段简单代码

```java
class Test1{
  public static void main(String[] args){
    String s1 = new String("hello");
    String s2 = "world";
  }
}
```

它的 class 文件常量池如下

```java
Constant pool:
   #1 = Methodref          #7.#16         // java/lang/Object."<init>":()V
   #2 = Class              #17            // java/lang/String
   #3 = String             #18            // hello
   #4 = Methodref          #2.#19         // java/lang/String."<init>":(Ljava/lang/String;)V
   #5 = String             #20            // world
   #6 = Class              #21            // Test1
   #7 = Class              #22            // java/lang/Object
   #8 = Utf8               <init>
   #9 = Utf8               ()V
  #10 = Utf8               Code
  #11 = Utf8               LineNumberTable
  #12 = Utf8               main
  #13 = Utf8               ([Ljava/lang/String;)V
  #14 = Utf8               SourceFile
  #15 = Utf8               Test1.java
  #16 = NameAndType        #8:#9          // "<init>":()V
  #17 = Utf8               java/lang/String
  #18 = Utf8               hello
  #19 = NameAndType        #8:#23         // "<init>":(Ljava/lang/String;)V
  #20 = Utf8               world
  #21 = Utf8               Test1
  #22 = Utf8               java/lang/Object
  #23 = Utf8               (Ljava/lang/String;)V
```

从 class 文件常量池可以看出只要是 `""` 括起来的文本字符串，都会进入 class 文件常量池。也可以理解为`""`括起来的都为字面量。

当 Test1 类被加载后（还没有到连接中的解析阶段），class 文件常量池中的东西会被存放到运行时常量池中，注意，此时还没有字符串对象生成。

JVM规范里Class文件的常量池项的类型，有两种东西：CONSTANT_Utf8、CONSTANT_String。后者是String常量的类型，但它并不直接持有String常量的内容，而是只持有一个index，这个index所指定的另一个常量池项必须是一个CONSTANT_Utf8类型的常量，这里才真正持有字符串的内容。

在Hotspot VM 中，运行时常量池里，CONSTANT_Utf8 在类加载过程中就全部创建出来，相当于一个指针指向一个 Symbol 类型的 c++ 对象，内容是跟 class 文件中相同格式的 utf8 字符串。而 **CONSTANT_String 是 lazy resolve 的**，在还没有 resolve 之前，JVM 将它的类型称为 JVM_CONSTANT_UnresolvedString，内容跟 class 文件常量池中一样只有一个 index。

从上面可以得知，对 HotSpot VM 来说，加载类的时候，这些字符串字面量会进入当前类的运行时常量池，不会进入全局字符串常量池，即 StringTable 中没有相应的引用，在堆中也没有相应的对象产生。



#### 那 CONSTANT_String 什么时候被 resolve ?

答案是遇到 ldc 指令。

ldc 指令的作用是将int、float或String类型的常量值从运行时常量池中推送至栈顶。比如下面的代码

```java
class Test2{
    public static void main(String[] args){
        String s = "hello";
    }
}
```

main 方法对应的字节码

```java
  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=2, args_size=1
         0: ldc           #2                  // String hello
         2: astore_1
         3: return
      LineNumberTable:
        line 3: 0
        line 4: 3
}
```

ldc 指令将字符串 hello 从运行时常量池中推送到栈顶，然后 astore_1 指令表示将这个字符串赋值给局部变量 s，接着就结束了。

既然 ldc 要将字符串从运行时常量池中推送到栈顶，所以当发现对应的 CONSTANT_String 是还没有解析的，就会进行解析，然后将解析之后的内容返回。**当遇到字符串类型常量，resolve 过程如果发现 StringTable 有了内容匹配的 java.lang.String 的引用，则会直接返回这个引用，否则将会在堆中创建一个对应内容的字符串对象，并将引用记录到 String Table 中，然后返回这个引用。**

**也就是说，我们代码中的字符串字面量在进行 resolve 之后都会在堆中有相应的对象，并且这个对象的的引用会被存放到 String Table 中。**



### 一些例子

在这之前，先了解一下 `java.lang.String.intern()`方法。在 JDK7 及以上，对一个字符串调用 intern 方法，如果常量池中已经有了这个字符串（通过 equals 比较）则直接返回常量池中的引用，如果没有，则将该字符串的引用保存到字符串常量池并且返回该引用，注意这个引用都是指向堆中的对象。

例子1：

```java
class Test1{
  public static void main(String[] args){
    String s1 = new String("hello");  //1
    String s2 = s1.intern();  //2
    System.out.println(s1 == s2); // false //3
  }
}
```

第一句对应的字节码

```java
0: new           #2           // class java/lang/String
3: dup
4: ldc           #3           // String hello
6: invokespecial #4           // Method java/lang/String."<init>":(Ljava/lang/String;)V
9: astore_1

```

new 是创建一个字符串对象实例，对其进行默认初始化，并将指向该实例的一个引用压入操作数栈顶，dup指令复制该实例引用到操作数栈顶。接着执行到 ldc 指令，这时将 #3对应的 CONSTANT_String 类型推到栈顶，发现他还没有解析，于是进行解析，解析过程中发现 StringTable 没有内容匹配的 java.lang.String 的引用，于是在堆中创建字符串 hello 的对象并将其存入字符串常量池中。接下来 invokespecial 执行对象构造函数，最后取出栈顶的引用储存到局部变量 s1 中。注意此时有两个 hello 对象，一个是执行 ldc 是创建的，它的引用驻留在字符串常量池中，一个是我们自己 new 的。

接下来执行第二句，s1.intern() 会查找字符串常量池中是否有 hello 字符串的引用，因为在执行 ldc 的时候 JVM 已经将 hello 对象驻留在字符串常量池中，也就是字符串常量中已经有内容为 hello 的对象，所以直接返回该引用，而且并不会将 s1 的引用放入字符串常量池，所以结果为 false。

[字节码更详细](https://github.com/zpffly/notebook/blob/master/jvm/Java字节码.md)

例子2：

```java
class Test2{
    static String s1 = "static"; //1
    public static void main(String[] args) {
        String s2 = new String("he") + new String("llo"); //2
        s2.intern();   //3
        String s3 = "hello";  //4
        System.out.println(s3 == s2);//输出是true。
    }
}
```

对于代码中出现的字面量 "static"，"he"，"llo"，"hello" 都会进入 class文件常量池，但是由于类加载阶段中 resolve 阶段是 lazy 的，所以不会创建实例，也就没有驻留字符串常量池。但是，对于 "static" 来说是特殊的。因为 JVM 在类加载阶段中的初始化阶段就会为静态变量指定初始值，也就是将 static 赋值给 s1，所以这时会创建"static"字符串对象，并且会保存一个指向它的引用到字符串常量池。对应的字节码指令如下

```java
  static {};
    descriptor: ()V
    flags: (0x0008) ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: ldc           #11                 // String static
         2: putstatic     #12                 // Field s1:Ljava/lang/String;
         5: return
      LineNumberTable:
        line 2: 0
}
```

注意 JVM 会静态变量初始化放到静态代码块中执行，而且是在类初始化时就会执行的。

接下来执行 main() 方法。

第二句： "he"，"llo" 进入字符串常量池过程同上个例子类似，对于两个字符串对象的相加，其内部是创建 StringBuilder 对象并且调用 append方法，最后调用StringBuilder对象的toString方法 new 出一个内容为hello的 String 对象，注意此时没有将这个对象的引用放入字符串常量池。

对于 jdk9来说字符串相加进行了优化。[相关链接](https://juejin.im/post/5b4be51e51882519a62f5835)

接下来第三句，因为字符串常量池中没有 hello 内容的字符串引用，所以 intern 方法将 hello 对象的引用保存到字符串常量池中。

接下来第四句，因为字符串常量池中有 hello 内容的字符串，所以直接返回引用。

所以第五句打印为 true。

整理自[Java 中new String("字面量") 中 "字面量" 是何时进入字符串常量池的? ](https://www.zhihu.com/question/55994121/answer/147296098)

