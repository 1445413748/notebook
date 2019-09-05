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