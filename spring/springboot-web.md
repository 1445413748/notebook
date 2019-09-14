## springboot 对静态资源的映射规则

在自动配置类 WebMvcAutoConfiguration 中，有对资源映射的处理

```java
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
	if (!this.resourceProperties.isAddMappings()) {
		logger.debug("Default resource handling disabled");
		return;
	}
	Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
	CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
	if (!registry.hasMappingForPattern("/webjars/**")) {
		customizeResourceHandlerRegistration(registry.addResourceHandler("/webjars/**")
				.addResourceLocations("classpath:/META-INF/resources/webjars/")
				.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
	}
    // staticPathPattern 为 /**
	String staticPathPattern = this.mvcProperties.getStaticPathPattern();
    //如果资源没有配置映射路径
	if (!registry.hasMappingForPattern(staticPathPattern)) {
//对访问 /** 下的文件进行映射
        customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
// 映射到 this.resourceProperties.getStaticLocations() 
                                             .addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
				.setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
	}
}

```



而 `this.resourceProperties.getStaticLocations()  ` 指向以下目录

```java
private static final String[] CLASSPATH_RESOURCE_LOCATIONS = 
{ "classpath:/META-INF/resources/", "classpath:/resources/", "classpath:/static/", "classpath:/public/" };
```

也就是说，对于没有映射的资源会到下面的目录去寻找

```java
classpath:/META-INF/resources/
classpath:/resources/
classpath:/static/
classpath:/public/
```



## Thymeleaf使用&语法

```java
@ConfigurationProperties(prefix = "spring.thymeleaf")
public class ThymeleafProperties {

	private static final Charset DEFAULT_ENCODING = StandardCharsets.UTF_8;
	// 前缀
	public static final String DEFAULT_PREFIX = "classpath:/templates/";
	// 后缀
	public static final String DEFAULT_SUFFIX = ".html";

```

thymeleaf  指定了页面的前缀为 classpath:/templates/ ，后缀为 .html。所以，只要将 HTML 页面放在指定的路径，thymeleaf 就会自动渲染。

[官方文档](https://www.thymeleaf.org/doc/tutorials/3.0/usingthymeleaf.pdf)



## spring MVC 自动配置

##### Spring MVC Auto-configuration

Spring Boot provides auto-configuration for Spring MVC that works well with most applications.

The auto-configuration adds the following features on top of Spring’s defaults:

- Inclusion of `ContentNegotiatingViewResolver` and `BeanNameViewResolver` beans.
  + 自动配置了 ViewResolver （视图解析器，根据方法解析器返回值得到视图对象，视图对象决定如何渲染）
  + `ContentNegotiatingViewResolver` : 组合所有视图解析器

- Support for serving static resources, including support for WebJars (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.2.0.BUILD-SNAPSHOT/reference/htmlsingle/#boot-features-spring-mvc-static-content))). 静态资源文件夹路径

- Automatic registration of `Converter`, `GenericConverter`, and `Formatter` beans.

  + `Converter` 转换器 

    比如将 controller 传过来的自定义类进行转换。

  + `Formatter`  格式化器

    比如日期转换： 2017/01/15  --->   Date

- Support for `HttpMessageConverters` (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.2.0.BUILD-SNAPSHOT/reference/htmlsingle/#boot-features-spring-mvc-message-converters)).

- Automatic registration of `MessageCodesResolver` (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.2.0.BUILD-SNAPSHOT/reference/htmlsingle/#boot-features-spring-message-codes)).

- Static `index.html` support.  静态首页访问

- Custom `Favicon` support (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.2.0.BUILD-SNAPSHOT/reference/htmlsingle/#boot-features-spring-mvc-favicon)).

- Automatic use of a `ConfigurableWebBindingInitializer` bean (covered [later in this document](https://docs.spring.io/spring-boot/docs/2.2.0.BUILD-SNAPSHOT/reference/htmlsingle/#boot-features-spring-mvc-web-binding-initializer)).

If you want to keep Spring Boot MVC features and you want to add additional [MVC configuration](https://docs.spring.io/spring/docs/5.2.0.RC2/spring-framework-reference/web.html#mvc) (interceptors, formatters, view controllers, and other features), you can add your own `@Configuration` class of type `WebMvcConfigurer` but **without** `@EnableWebMvc`. If you wish to provide custom instances of `RequestMappingHandlerMapping`, `RequestMappingHandlerAdapter`, or `ExceptionHandlerExceptionResolver`, you can declare a `WebMvcRegistrationsAdapter` instance to provide such components.

如果想保持 springboot MVC 的特性，并且增加一些额外的配置，如拦截器、视图解析器等，可以增加自己的配置类，而且这个类的类型为`WebMvcConfigurer`，不能在类上添加 `@EnableWebMvc` 注解。

比如添加自动跳转配置

```Java
@Configuration
public class MyMVCConfig implements WebMvcConfigurer {

    // 增加路径映射，直接映射，这样就不用在 controller 写跳转方法
    @Override
    public void addViewControllers(ViewControllerRegistry registry) {
        registry.addViewController("/test")
                .setViewName("test");
    }
}
```

当在配置类添加了 `@EnableWebMvc` 

```java
//导入 DelegatingWebMvcConfiguration
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
}
```

```java
@Configurationpublic 
//继承 WebMvcConfigurationSupport
class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {   
    private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
```

而 WebMvcAutoConfiguration 生效需要的条件

```java
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
// 没有WebMvcConfigurationSupport 才生效
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {

```

