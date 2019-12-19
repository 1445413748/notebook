## Bean的作用域

+ singleton

  单例，指一个 `Bean` 容器只存在一份

+ prototype

  每次请求（使用）创建新的实例，destory 方式不生效，请求结束后实例被等待被垃圾回收器回收

+ request

  每次 `HTTP` 请求会创建一个实例，并且只在当前请求有效，**当请求结束后，该对象的生命周期即告结束**

+ session

  每次 `HTTP` 请求会创建新实例，并且只在当前 `session` 有效

+ global session

  类似与 `Session`，不过仅仅在基于 `portlet` 的 web 应用有效



## Bean生命周期

### initialization 和 destroy

`initialization` 和 `destroy`  分别对应  `Bean` 设置好属性之后和销毁之前，有多种方式执行 `initialization` 和 `destroy` 。

#### 1、在 bean 配置文件指定 init-method 和 destroy-method 方法

这是 bean ：

```java
public class BeanLifeCycle {

    public void start(){
        System.out.println("执行 init()...");
    }

    public void stop(){
        System.out.println("执行 destroy()...");
    }
}
```

在 xml 中的配置：

```xml
<bean class="com.example.demo.bean.BeanLifeCycle" id="beanLifeCycle" init-method="start" destroy-method="stop"/>
```

**这种自定义在 xml 配置的方法可以抛异常但是不能有参数**

使用的Test类：

```java
class BeanLifeCycleTest {

    ClassPathXmlApplicationContext context;

    @BeforeEach
    public void before(){
        context = new ClassPathXmlApplicationContext("classpath:spring-bean.xml");
        // 启动ioc容器
        context.start();
    }

    @AfterEach
    public void after(){
        // 关闭ioc容器
        context.close();
    }


    @Test
    public void test1(){
        Object beanLifeCycle = context.getBean("beanLifeCycle");
    }
    
}
```

结果：

```
执行 init()...
执行 destroy()...
```



#### 2、实现 initializingBean 和 DisposableBean 接口

```java
public class BeanLifeCycle implements InitializingBean, DisposableBean {

    public void start(){
        System.out.println("执行 init()...");
    }

    public void stop(){
        System.out.println("执行 destroy()...");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("InitializingBean的afterPropertiesSet被执行");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("DisposableBean的destroy被执行");
    }
}
```

在 `xml` 里配置 `bean`，注意，没有配置 `init-method` 和 `destroy-method`

```xml
<bean class="com.example.demo.bean.BeanLifeCycle" id="beanLifeCycle"/>
```

执行结果：

```
InitializingBean的afterPropertiesSet被执行
DisposableBean的destroy被执行
```

#### 三、使用 @PostConstruct 和 @PreDestroy

```java
public class BeanLifeCycle{
    @PostConstruct
    public void initPostConstruct(){
        System.out.println("使用PostConstruct init");
    }

    @PreDestroy
    public void preDestroy(){
        System.out.println("使用PreDestroy destroy");
    }
}
```

为了使注解生效，需要在在 xml 中配置 `context:annotation-config`

运行结果：

```
使用PostConstruct init
使用PreDestroy destroy
```

#### 四、三者优先级

同时配置三者，然后运行上面的test1得到结果

```
使用PostConstruct init
InitializingBean的afterPropertiesSet被执行
执行 init()...
使用PreDestroy destroy
DisposableBean的destroy被执行
执行 destroy()...
```

从结果可以看出，执行的顺序是：注解 -> 实现接口 -> 配置init-method 和 destroy-method



### *Aware 接口

`spring` 提供了一些以 `Aware` 结尾的接口，实现 `Aware` 接口的 `bean` 在初始化后可以获取 `Spring` 相应的资源，并且操作这些资源。

常见：

- ApplicationContextAware：获得ApplicationContext对象,可以用来获取所有Bean definition的名字。
- BeanFactoryAware：获得BeanFactory对象，可以用来检测Bean的作用域。
- BeanNameAware：获得Bean在配置文件中定义的名字。
- ResourceLoaderAware：获得ResourceLoader对象，可以获得classpath中某个文件。
- ServletContextAware：在一个MVC应用中可以获取ServletContext对象，可以读取context中的参数。
- ServletConfigAware： 在一个MVC应用中可以获取ServletConfig对象，可以读取config中的参数。

```java
public class BeanLifeCycle implements InitializingBean,DisposableBean,BeanNameAware, ApplicationContextAware {
    String name;
    ApplicationContext context;
    int number;

    // 普通的setter方法
    public void setNumber(int num){
        System.out.println("执行BeanLifeCycle的setNumber");
        number = num;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("执行InitializingBean的afterPropertiesSet");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("执行DisposableBean的destroy");
    }

    @Override
    public void setBeanName(String s) {
        System.out.println("执行BeanNameAware的setBeanName");
        // 得到bean的name
        name = s;
    }


    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.println("执行ApplicationContextAware的setApplicationContext");
        // 得到上下文信息
        context = applicationContext;
    }
}
```

运行 test1 得到：

```
执行BeanLifeCycle的setNumber
执行BeanNameAware的setBeanName
执行ApplicationContextAware的setApplicationContext
执行InitializingBean的afterPropertiesSet
执行DisposableBean的destroy
```



### BeanPostProcessor

*Aware 是针对某个实现了这些接口的`Bean` 定制初始化过程，Spring 还提供了针对整个容器中所有或者某些 `bean` 定制初始化过程，只需要提供一个实现BeanPostProcessor接口的类即可。

该接口提供了两个函数：

+ postProcessBeforeInitialzation( Object bean, String beanName )

  当前正在初始化的 `bean` 对象会被传递进来，我们i可以对其进行处理，因为这个函数会先于 `InitializingBean` 执行，所以被称为前置处理。

+ postProcessAfterInitialzation( Object bean, String beanName ) 
  当前正在初始化的bean对象会被传递进来，我们就可以对这个bean作任何处理。 
  这个函数会在InitialzationBean完成后执行，因此称为后置处理。



```java
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("执行BeanPostProcessor的postProcessBeforeInitialization");
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("执行BeanPostProcessor的postProcessAfterInitialization");
        return bean;
    }
}
```

要将 `MyBeanPostProcessor` 添加到 `IOC` 容器

```xml
<bean class="com.example.demo.bean.MyBeanPostProcessor" id="processor"/>
```

执行结果：

 ```
执行BeanLifeCycle的setNumber
执行BeanNameAware的setBeanName
执行ApplicationContextAware的setApplicationContext
执行BeanPostProcessor的postProcessBeforeInitialization
执行InitializingBean的afterPropertiesSet
执行BeanPostProcessor的postProcessAfterInitialization
执行DisposableBean的destroy
 ```



### 生命周期

![](img/life.jpg)

#### 1. 实例化Bean

对于 `BeanFactory` 容器，当客户向容器请求一个尚未初始化的 `bean` 时，容器会调用 `createBean` 进行实例化。

如果容器是 `ApplicationContent`（其实是BeanFactory子类），则容器启动后会实例化所有的 `bean`（懒加载除外）。

注意，此时只是简单的初始化，容器通过获取 `BeanDefinition` 对象（存储了 Bean 的元信息）中的信息进行实例化，实例化的对象被包装在 `BeanWrapper` 中，还没有进行依赖注入。



#### 2.设置对象属性

根据 `BeanDefinition` 中的信息进行依赖注入，注入是通过 `BeanWrapper` 提供的接口。



#### 3. 注入Aware接口

`Spring` 会检测对象实现了哪些 `Aware` 接口，将对应的 `Aware` 实例注入



#### 4. 执行BeanPostProcessor的前置处理

执行前置处理 `postProcessBeforeInitialzation( Object bean, String beanName )`



#### 5. 执行 InitializingBean

当属性注入完毕并且执行完前置处理后，会执行 `InitializingBean` 接口的 `afterPropertiesSet()`



#### 6. 执行BeanPostProcessor的后置处理

执行后置处理 `postProcessAfterInitialzation( Object bean, String beanName )`



#### 7. 执行DisposableBean

当准备关闭 `ioc` 容器或者不再需要 `bean` 时，会执行 `DisposableBean.destroy()`





## Bean自动装配

### 扫描注解

为了使下面的使用的注解生效，应该先使用 `@ComponentScan` 指明扫描哪里的注解，可以通过包名指定，也可以通过类 .class 指定

也可以在`xml`文件中配置，如下：

```xml
<context:component-scan base-package="com.example.demo"/>
```

### 使用注解 @Autowired

`@Autowired` 默认是根据注入类的类型在 `ioc` 容器中寻找合适的 `bean`，如果该类型存在多个 `bean` ，则需要进行处理。

因为 `@Autowired` 是由 `BeanPostProcessor` 处理的，所以不能在我们自己的 `BeanPostProcessor` 或 `BeanFactorPostProcessor` 类型应用（造成循环依赖），可以通过 `XML` 或者 `@Bean` 注解加载 

Good 类：

```java
@Repository
public class Good {
    public void save(){
        System.out.println("good save in dao");
    }
}
```



#### 一、使用在类属性上

```java
@Service
public class GoodService {

    @Autowired
    private Good good;

    public void save(){
        System.out.println("save in service");
        good.save();
    }
}
```

测试

```java
@Test
void tes1() {
    GoodService goodService = (GoodService)context.getBean("goodService");
    goodService.save();
}
```

输出

```
save in service
good save in dao
```



#### 二、使用在方法上

```java
@Service
public class GoodService {
    
    private Good good;
    
    @Autowired
    public void setGood(Good good){
        this.good = good;
    }

    public void save(){
        System.out.println("save in service");
        good.save();
    }
}
```

#### 三、使用在构造器上

```java
@Service
public class GoodService {

    private Good good;

    @Autowired
    public GoodService(Good good){
        this.good = good;
    }

    public void save(){
        System.out.println("save in service");
        good.save();
    }

}
```

#### 四、使用在数组及Map

当 `@AutoWired` 使用在数组上，会自动注入`ioc` 中所有数组指定类型的bean

```java
// 接口
public interface BeanInterface {
}

// 实现接口的两个类
@Component
public class BeanImplOne implements BeanInterface {
}

@Component
public class BeanImplTwo implements BeanInterface {
}

```

BeanInvoker：

```java
@Component
public class BeanInvoker {

    @Autowired
    List<BeanInterface> list;  //类型为BeanInterface，如果指定类型相当于找list类型的bean

    public void printList(){
        if (list != null){
            for (BeanInterface bean : list)
                System.out.println(bean.getClass().getName());
        }else {
            System.out.println("list is null");
        }
    }
}
```

我们通过 `BeanInvoker` 打印 list

```java
@Test
void test2(){
    BeanInvoker invoker = (BeanInvoker) context.getBean("beanInvoker");
    invoker.printList();
}
```

结果

```
com.example.demo.bean.BeanImplOne
com.example.demo.bean.BeanImplTwo
```



使用在 `Map` 注意 `key`的类型为 `String` ，对应为 `bean` 的 id，后面则是我们指定的类型

```java
@Component
public class BeanInvoker {

    @Autowired
    Map<String, BeanInterface> map;

    public void printMap(){
        if (map != null){
            for (Map.Entry<String, BeanInterface> entry : map.entrySet()){
                System.out.println(entry.getKey() + "-->" + entry.getValue());
            }
        }else {
            System.out.println("map is null");
        }
    }
}
```

执行`printMap` 结果

```
beanImplOne-->com.example.demo.bean.BeanImplOne@5909ae90
beanImplTwo-->com.example.demo.bean.BeanImplTwo@4489f60f
```

如果希望注入的`Bean` 是有序的，可以使用 `@Order` 注释在我们相应类型的类上即可

















## Reference

[Spring中Bean的生命周期是怎样的？ - 大闲人柴毛毛的回答 - 知乎]( https://www.zhihu.com/question/38597960/answer/247019950)

[Spring中bean的作用域与生命周期](https://zhuanlan.zhihu.com/p/44875302)