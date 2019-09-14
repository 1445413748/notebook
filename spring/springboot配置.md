## 配置文件注入

### @Value 和 @ConfigurationProperties 

 @ConfigurationProperties(prefix = "name") 告诉 springboot 将本类的属性和配置文件的相关配置进行绑定, prefix 指定了前缀名

```java
@Component  //注册到容器中
@ConfigurationProperties("person")  //前缀为 person
@Data
public class Person {
    private String name;
    private Integer age;
    private Boolean student;
    private Date birthday;

    private Map<String, Object> map;
    private List<Object> list;
    private Dog dog;
}
```

在配置文件中（yaml 文件）

```yaml
person:
  name: zpffly
  age: 20
  student: true
  birthday: 1998/04/25
  map: {k1: v1, k2: v2}
  list:
    - l1
    - l2
  dog:
    name: fafa
```

properties文件中

```properties
#person配置
person.name=张三
person.age=20
person.birthday=2018/11/14
person.student=false
person.dog.name=dog
person.list=a,b,c
person.map.k1=v1
person.map.k2=v2
```

**使用 @value("valueName") 也可以注入属性值**

比如使用上面的 properties 文件，如果我要注入 person name属性值，则

```java
@Value("${person.name}")
private String name;
```



#### @Value 和 @ConfigurationProperties 区别

|                | @ConfigurationProperties |     @value      |
| :------------: | :----------------------: | :-------------: |
|      功能      |  批量注入配置文件中属性  |   一个个指定    |
|    松散绑定    |           支持           |     不支持      |
|      SpEL      |          不支持          |      支持       |
| JSR303数据校验 |           支持           | springboot2支持 |
|  复杂类型封装  |           支持           |     不支持      |

复杂类型封装：

比如对于 Person 的 map 字段，如果通过下面的方式注入则会出错。

```java
@Value("{person.map}")
private Map<String, Object> map;
```

### @PropertySource 和 @ImportResource

#### @PropertySource

@ConfigurationProperties默认从全局配置文件中获取值，为了不让配置文件变得臃肿，可以将不同配置独立成多个配置文件，而通过 @PropertySource 就可以从指定的位置加载配置文件。

将 Person 的配置独立出来成 person.properties

```yam
person.name=李四
person.age=20
person.birthday=2018/11/14
person.student=false
person.dog.name=dog
person.list=a,b,c
person.map.k1=v1
person.map.k2=v2
```

然后添加注解 @PropertySource(value = {"classpath:person.properties"})

```java
@Component
//从类路径下加载person.properties
@PropertySource(value = {"classpath:person.properties"}) 
@ConfigurationProperties("person")
@Data
@Validated
public class Person {
    //@Value("person.name")
   // @Email
    private String name;
    private Integer age;
    private Boolean student;
    private Date birthday;

//    @Value("${person.map}")  //error
    private Map<String, Object> map;
    private List<Object> list;
    private Dog dog;
}
```

#### @ImportResource

作用：导入 spring 配置文件。让配置文件生效。

在类路径下创建 beans.xml 文件，在里面配置 Course 类

```xml
<bean id="course" class="com.scnu.zpffly.Demo2.Course">
    <property name="name" value="java"/>
</bean>
```

然后查看 IOC 容器中是否有 course 这个 javaBean

```java
public class CourseTest {

    @Autowired
    private ApplicationContext ioc;

    @Test
    public void test1(){
        System.out.println(ioc.containsBean("course")); //false
    }

}
```

接下来在启动类添加注解`@ImportResource(locations = {"classpath:beans.xml"})`，打印的结果为 true。

#### springboot 推荐使用全注解方式添加组件 

用配置类代替 spring 配置文件。

```java
// @Configuration 标记这个类为配置类
@Configuration
public class CourseConfig {

    // @Bean 将返回的类添加到容器中
    @Bean
    public Course course(){
        Course course = new Course();
        course.setName("springboot");
        return course;
    }
}
```



## Profile

### 1、多 Profile 文件

多个配置文件的名字可以是 application-{profile}.properties/yml，比如 application-dev.properties

默认使用 application.properties 的配置。

### 2、yml 多文档方式

yml 三个横线 `---` 将文档分割为多个文档块，不同文档块可以是不同的环境，比如：

```yaml
profile:
  msg: 这是第一文档块(默认环境)
spring:
  profiles:
    active: dev   # 指定 dev 环境
---
profile:
  msg: 这是 dev 环境
spring:
  profiles: dev  #指定环境名

---
profile:
  msg: 这是 prod 环境

spring:
  profiles: prod
```



### 3、激活指定的 Profile

1. 在配置文件中指定 spring.profile.active={profile}，比如指定 application-{profile}.properties，则可以配置 spring.profile.active=dev
2. 命令行：  --spring.profiles.active=dev;
3. 虚拟机参数：-Dspring.profiles.active=dev

上面优先级从低到高

### 例子

```java
/**
 * 多环境测试类，测试使用哪个环境
 */
@Component
@ConfigurationProperties("profile")
@Data
public class Profile {
    private String msg;
}
```

在主配置文件配置msg

```properties
profile.msg=使用的是主配置文件
```

在 dev 配置文件配置 msg

```properties
profile.msg=使用的是主配置文件
```

在测试方法打印 profile

```java
public class ProfileTest {

    @Autowired
    private Profile profile;

    @Test
    public void test(){
        System.out.println(profile);
    }
}
```

结果为

```java
Profile(msg=使用的是主配置文件)
```

在主配置文件 application.properties 配置环境

```properties
spring.profile.active=dev
```

结果为

```java
Profile(msg=使用的是 dev 测试环境)
```



## 配置文件加载位置

springboot 启动会扫描以下位置的 application.properties 或者 application.yml 作为默认配置文件。

1. -file:./config/
2. -file:./（当前文件的根目录）
3. -classpath:/config/
4. -classpath:/

优先级从高到低，高优先级配置覆盖低优先级的配置。springboot 会从这四个加载全部配置文件，几个文件互补。

**通过 spring.config.location可以改变默认的配置文件位置**

**如果项目已经打包好，可以通过命令行参数的形式，在启动的时候指定配置文件的新位置。指定的文件和项目中的配置文件形成互补配置。**



## 自动配置原理

[配置文件属性参照](https://docs.spring.io/spring-boot/docs/2.2.0.BUILD-SNAPSHOT/reference/htmlsingle/#common-application-properties)

### 自动配置

​    springboot 启动时加载主配置类，开启了自动配置功能  `@EnableAutoConfiguration`

#### @EnableAutoConfiguration 

作用：开启自动配置

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
	Class<?>[] exclude() default {};
	String[] excludeName() default {};
}

```

@**AutoConfigurationPackage**：自动配置包

```java
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}
```

在`@AutoConfigurationPackage`中导入组件 `AutoConfigurationPackages.Registrar.class`。

在导入的组件中通过下面的方法将主配置类（@SpringbootApplication标注的类）的所在包及子包下面的所有组件扫描到 spring 容器。

```java
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
			register(registry, new PackageImport(metadata).getPackageName());
		}

```



在 @EnableAutoConfiguration 还有 `@Import(AutoConfigurationImportSelector.class)`。

**AutoConfigurationImportSelector**：导入哪些组件的选择器。在这个类的中

```java
@Override
public String[] selectImports(AnnotationMetadata annotationMetadata) {
	if (!isEnabled(annotationMetadata)) {
		return NO_IMPORTS;
	}
	AutoConfigurationMetadata autoConfigurationMetadata = 		 AutoConfigurationMetadataLoader.loadMetadata(this.beanClassLoader);
	AutoConfigurationEntry autoConfigurationEntry = 			getAutoConfigurationEntry(autoConfigurationMetadata,annotationMetadata);
		return StringUtils.toStringArray(autoConfigurationEntry.getConfigurations());
}
```

selectImports 的作用是收集要导入的配置类。

selectImports 中 getAutoConfigurationEntry 方法：得到应该被导入的自动配置

```java
	/**
	 * Return the {@link AutoConfigurationEntry} based on the {@link AnnotationMetadata}
	 * of the importing {@link Configuration @Configuration} class.
	 * @param autoConfigurationMetadata the auto-configuration metadata
	 * @param annotationMetadata the annotation metadata of the configuration class
	 * @return the auto-configurations that should be imported
	 */List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
	protected AutoConfigurationEntry getAutoConfigurationEntry(AutoConfigurationMetadata autoConfigurationMetadata,
			AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
		}
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
        // 获取所有通过META-INF/spring.factories配置的, 此时还不会进行过滤和筛选
		// KEY为 ： org.springframework.boot.autoconfigure.EnableAutoConfiguration
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
        //过滤，去重
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
		configurations = filter(configurations, autoConfigurationMetadata);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return new AutoConfigurationEntry(configurations, exclusions);
	}
```



在 getCandidateConfigurations 方法中，通过指定类的

```java
List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),      getBeanClassLoader());
```

获取获取候选的自动配置列表。

这里的 getSpringFactoriesLoaderFactoryClass() 返回的是 EnableAutoConfiguration.class。

```java
protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;}
```

loadFactoryNames() ：

1. 扫描所有 jar 包类路径下的 META-INF/spring.factories
2. 把扫描到的文件内容包装成 properties 对象。
3. 从 properties 中获取到 EnableAutoConfiguration.class 对应的值，添加到容器中。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration,\
org.springframework.boot.autoconfigure.batch.BatchAutoConfiguration,
```

每一个类对应一个配置类

以

```java
//表示为配置类
@Configuration
//判断当前容器中有没有这些类
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
//启动指定类的 ConfigurationProperties 功能
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class, DataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {
    //省略
}
```



配置文件能配置的 spring.datasource 的属性参考这个类中的属性

```java
//从配置文件获取指定的值和 bean 属性进行绑定
@ConfigurationProperties(prefix = "spring.datasource") //指定前缀名spring.datasource
public class DataSourceProperties implements BeanClassLoaderAware, InitializingBean {
}
```

比如 DataSourceProperties 中有下面的属性

```Java
	/**
	 * JDBC URL of the database.
	 */
	private String url;

	/**
	 * Login username of the database.
	 */
	private String username;

	/**
	 * Login password of the database.
	 */
	private String password;

```

就可以在配置文件中配置

> spring.datasource.username=zpffly



**在 DataSourceAutoConfiguration() 会配置一些组件，而组件的来源则是我们在配置文件中配置的值。**