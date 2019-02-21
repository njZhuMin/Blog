---
title: Spring框架-IoC
date: 2016-06-01 09:31:16
tags: [spring,IoC]
categories:
- framework
- Spring
---

> Spring系列文章：
> https://zhum.in/blog/categories/frame/Spring/

> 项目代码：
> https://github.com/njZhuMin/SpringDemo

- - -
# 框架
## 框架的特点
- 半成品
- 封装了特定的处理流程和控制逻辑
- 成熟的、不断升级改进的软件

## 框架与类库的区别
- 框架一般是封装了逻辑、高内聚的，类库则是松散的工具组合
- 框架专注于某一领域，类库则是更通用的

## 为什么使用框架
1. 软件系统日趋复杂
2. 重用度高，开发效率和质量提高
3. 软件设计人员要专注于对领域的了解，使需求分析更充分
4. 易于上手，快速解决问题

<!-- more -->

# Spring IoC容器
## 面向接口编程的特点
- 结构设计中，分清层次及调用关系，每层只向外（上层）提供一组功能借口，各层间仅依赖接口而非实现类
- 接口实现的变动不影响各层间的调用，这一点在公共服务中尤为重要
- `面向接口编程`中的`接口`是用于隐藏具体实现和实现多态性的组件

## 控制反转（IoC）
1. IOC（控制反转），控制权的转移，应用程序本身不负责依赖对象的创建和维护，而是由外部容器负责创建和维护

2. DI（依赖注入）是其一种实现方式

3. 目的：创建对象并且组装对象之间的关系
IOC在初始化的时候会创建一系列的对象，对象之间的关系通过依赖注入组织起来

## Bean容器的初始化
- 初始化的基础
`org.springframework.beans`、`org.springframework.context`
BeanFactory 提供配置结构和基本功能，加载并初始化Bean
ApplicationContext保存了Bean对象并在Spring中被广泛使用

- 初始化的方式：ApplicationContext
1. 本地文件
```java
FileSystemXmlApplicationContext context = new FileSystemXmlApplicationContest("path/to/appContext.xml");
```
2. Classpath
```java
ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("classpath:spring-context.xml");
```
3. Web应用中依赖servlet或Listener
```xml
<listener>
	<listener-class>
		org.springframework.web.context.ContextLoaderListener
	</listener-class>
</listener>
<servlet>
	<servlet-name>context</servlet-name>
	<servlet-class>
		org.springframework.web.context.ContextLoaderServlet
	</servlet-class>
</servlet>
```

## Spring注入
Spring是指在启动Spring容器加载bean配置的时候，完成对变量的赋值行为。常用注入方式有`设值注入`和`构造注入`

- 设值注入
不需要显示地调用set方法，会根据xml的相关配置自动进行调用set方法。

 实质上就是Spring利用属性或成员变量的set方法进行注入（在serviceImpl中要定义injectionDAO，并设置set方法）。其中property里面的name是需要注入参数的成员变量的名称，ref是注入参数引入bean的名称

- 构造注入
在spring-injection.xml中的bean中要写`<constructor-arg name="injectionObj" ref="injectionObj">`。

 与设置注入一样，但是在在serviceImpl中要定义构造函数

> 注意：参数的名称必须保持一致

## Bean的作用域
- singletion单例：bean容器只有唯一的对象（默认模式）

- prototype：每次请求会创建新的实例，destory方式不生效

- request：对于request创建新的实例，只在当前request内有效

- session：对于session创建新的实例，只在当前session内有效

- global session：基于portlet（例如单点登录的范围）的web中有效，如果在web中，范围则是同一个session

## Bean的生命周期
1.定义

2.初始化：
- 实现org.springframework.beans.foctory.InitializingBean接口，覆盖afterPropertiesSet方法。系统会自动查找afterPropertiesSet方法，执行其中的初始化操作
- 配置init-method
在配置文件中设置bean中的`<init-method="init">`属性，那么在初始化过程中Spring就会调用相应class指定类的init()方法进行初始化工作

3.使用

4.销毁
- 实现org.springframework.beans.foctory.DisposableBean接口，覆盖destory方法。
- 配置destory-method

> 配置全局初始化、销毁方法

> 在配置文件的`<beans>`标签中配置`default-init-method="init"`和`default-init-method="destroy"`属性。

- 执行顺序：
1. 当三种方式同时使用时，全局（默认的）初始化销毁方法会被覆盖；

2. 实现接口的初始化/销毁方式会先于配置文件中的初始化/销毁方式执行；

3. 若仅有全局初始化即使没有以上三种初始化方法也是可以编译执行的，但是有另两种初始化销毁方法时，必须存在相应的方法，否则编译报错

## Aware
Spring中提供了一些以`Aware`结尾的接口，实现了`Aware`接口的bean在被初始化之后可以获取相应资源。

- 通过Aware接口，可以对Spring相应资源进行操作（一定要慎重）

- 对为Spring进行简单的扩展提供了方便的入口

## Bean的自动装配（Autowiring）
- no：不做任何操作，Bean依赖必须通过使用ref元素定义。这是默认的配置

- byName：根据`属性名`自动装配，设值注入：
`<bean id="beanName" class="beanClass" ></bean>`

- byType：根据`属性类型`自动装配，相同类型多个会抛出异常，设值注入:
`<bean class="beanClass" ></bean>`

- constructor：与`byType`方式类似，不同之处是构造注入，用于自动匹配构造器参数:
`<bean class="beanClass" ></bean>`

> 使用`byType`方式时，如果存在多个类型的Bean，会抛出`不能使用byTpye装配`异常

> 使用`byName`方式时，如果存在相同id的Bean，那么IoC容器将启动失败并抛出异常

> 使用`constructor`方式时，如果找不到对应的构造器，那么将抛出异常

## Resource
Resource是Spring中针对于资源文件的统一接口。具体实现类有：
- UrlResource：URL对应的资源，根据一个URL地址即可构建
- ClassPathResource: 获取类路径下的资源文件
- FileSystemResource：获取文件系统里面的资源
- ServletContextResource：ServletContext封装的资源，用于访问ServletContext环境下的资源
- InputStreamResource：针对于输入流封装的资源
- ByteArrayResource：针对于字节数组封装的资源

其中ResourceLoader传入的路径可以由以下四种前缀：
1. classpath：com/myapp/config.xml（从classpath加载）
2. file:/data/config.xml（从文件系统中通过URL加载）
3. http:/myserver/logo.png（从URL加载）
4. /data/config.xml（依赖于ApplicationContext）

# Spring注解
## Bean的注册
默认情况下，类被自动发现并注册bean的条件是：使用`@Component`、`@Repository`、`@Service`、`@Controller`注解或者使用`@Component`的自定义注解。

在`Spring配置文件`中配置`component-scan`标签：
```xml
<beans>
	<context:component-scan base-package="com.example">
		<context:include-filter type="regex"
			expression=".*Stub.*Repository" />
		<context:exclude-filter type="annotation"
			expression="org.springframework.stereotype.Repository" />
	</context:component-scan>
</beans>
```

- 扫描过程中组件被自动检测，Bean的Name由`BeanNameGenerator`生成。可以通过实现`BeanNameGenerator`接口并包含无参构造器自定义命名策略。同时增加配置：
```xml
<beans>
	<context:component-scan base-package="com.example"
		name-generator="com.example.MyNameGenerator" />
</beans>
```

- Spring的Bean容器默认的范围是`Singleton`，可以通过实现`ScopeMetadataResolver`接口并提供无参构造器自定义scope策略。如我们需要为每一个线程分配独立的Bean容器。同时增加配置：
```xml
<beans>
	<context:component-scan base-package="com.example"
		scope-resolver="com.example.MyScopeResolver" />
</beans>
```

- 可以使用`scope-proxy`标签为Bean指定代理，可选属性有：`no`、`interfaces`和`targetClass`：
```xml
<beans>
	<context:component-scan base-package="com.example"
		scope-proxy="interfaces" />
</beans>
```

## @Required
`@Required`注解作用于Bean属性的setter方法，表示受影响的Bean属性必须在配置时被填充，通过Bean定义或AutoWire一个明确的属性值。

## @AutoWired
`@AutoWired`可以被用于setter方法、构造器或成员变量。默认情况下，如果找不到合适的bean将会导致autowiring失败并抛出异常。
> 可以使用`@AutoWired(required=false)`避免异常，但使用的时候必须判断对象是否为空。

每个类只能有一个构造器可以被标记为`@AutoWired(required=true)`，常用`@Required`注解代替。

`@AutoWired`注解可以注解其他常见的依赖性接口，比如`BeanFactory`、`ApplicationContext`、`Environment`、`ResourceLoader`、`ApplicationEventPublisher`和`MessageSource`等。

> 可以添加`@AutoWired`注解给需要该类型的数组的字段或方法，以提供`ApplicationContext`中的所有特定类型的bean。

```java
private Set<MovieCatalog> movieCatalogs;

@AutoWired
public void setMovieCatalogs(Set<MovieCatalog> movieCatalogs) {
	this.movieCatalogs = movieCatalogs;
}
```

> `@AutoWired`注解也可以用于装配Key为String的Map。如果希望数组有序，只需要bean实现`org.springframework.core.Ordered`接口。

```java
private Map<String, MovieCatalog> movieCatalogs;

@AutoWired
public void setMovieCatalogs(Map<MovieCatalog> movieCatalogs) {
	this.movieCatalogs = movieCatalogs;
}
```

> `@AutoWired`注解是由`Spring BeanPostProcessor`处理的，所以不能在自己的`BeanPostProcessor`或`BeanFactoryPostProcessor`类型应用这些注解，必须通过XML或`@Bean`注解完成加载。

注意：
`@Order`注解作用于bean上面指定序号，可以实现在List中对其排序，但是对Map无效。

## @Qualifier
注解可以按类型自动装配可能多个bean实例的情况，可以使用Spring的`@Qualifier`注解缩小范围（或指定唯一），也可以用于指定单独的构造器参数或方法参数。

`@Qualifier`可以注解集合类型的变量。

需要在xml文件中配置`<qualifier value="uniqueName">`标签。

## @Resource
如果通过独特的名称定义来识别特定的目标（这是一个与所声明的类型是无关的匹配过程），即使技术上能够通过`@Qualifier`注解指定bean的名称，但遵循JSR-250标准应使用`@Resource`注解。

因语义差异，集合或Map类型的bean无法通过`@AutoWired`来注入，因为没有类型匹配到这样的bean。为这些bean使用`@Resource`注解，通过唯一名称引用集合或Map的bean。

> `@AutoWired`注解适用于fields，constructors，multi-argument methods这些允许在参数级别使用`@Qualifier`注解缩小范围的情况。
> `@Resource`注解适用于成员变量、只有一个参数的setter方法，所以在目标是构造器或一个多参数的方法时，最好的方式是使用`@Qualifier`注解。

允许自定义`Qualifier`注解，需要在自定义的注解上使用`Qualifier`注解。

# 基于Java容器的注解
## @Bean
`@Bean`注解表示一个用于配置和初始化一个由Spring IoC容器管理的新对象的方法，类似于在xml文件中配置的`<bean>`标签。

可以在Spring的`@Component`注解的类中使用`@Bean`注解任何方法，但通常在`@Configuration`注解的类中使用。

例如，以下注解和xml配置是等价的：
```java
@Configuration
public class AppConfig {
	@Bean
	public MyService myService() {
		return new MyServiceImpl();
	}
}
```
```xml
<beans>
	<bean id="myService" class="com.example.service.MyServiceImpl" />
</beans>
```

## @ImportResource和@Value
与在xml文件中`context:property-placeholder>`标签中配置`location`属性相同，也可以使用`@ImportResource`注解来设置配置文件路径，使用`@Value`注解获取资源路径中的值（EL表达式格式）。

## 基于泛型的自动装配
定义一个带有泛型的接口和两个实现类：
```java
public interface Store<T> {
}

public class StringStore implements Store<String> {
}

public class IntegerStore implements Store<Integer> {
}
```
使用注解配置Spring自动装配：
```java
@Configuration
public class StoreConfig {

    @Autowired
    @Qualifier("stringStore")
    private Store<String> s1;

    @Autowired
    @Qualifier("integerStore")
    private Store<Integer> s2;

    @Bean
    public StringStore stringStore() {
        return new StringStore();
    }

    @Bean
    public IntegerStore integerStore() {
        return new IntegerStore();
    }

    @Bean
    public Store stringStoreTest() {
        System.out.println("s1: " + s1.getClass().getName());
        System.out.println("s2: " + s2.getClass().getName());
        return new StringStore();
    }
}
```
测试泛型：
```java
@Test
public void testG() {
    StringStore store = super.getBean("stringStoreTest");
}
```

# JSR标准注解
## JSR-250 @Resource
Spring支持使用JSR-250规范的`@Resource`注解的变量或setter方法。`@Resource`有一个name属性，默认Spring解释该值作为被注入bean的名称。如果没有显式的指出`@Resource`的name值，默认名称从属性名或者setter方法名得出。

注解提供的名字被解析为一个bean的名称，是由`ApplicationContext`中的`CommonAnnotationBeanPostProcessor`发现并处理的。
`CommonAnnotationBeanPostProcessor`不仅能识别生命周期注解`@Resource`，还引入`@PostConstruct`和`@PreDestroy`注解支持初始化回调和销毁回调，前提是`CommonAnnotationBeanPostProcessor`是Spring的`ApplicationContext`中注册的。

## JSR-330 @Inject @Named
`@Inject`注解等效于`@AutoWired`，可以适用于类、属性、方法和构造器。
`@Named`与`@Component`是等效的，可以对使用特定名称的bean进行以来注入。

`@Named`和`@Inject`一起使用： @Named可以放在类上，@Named("beanName")放在setter方法的形参前，@Inject可以放在变量或setter上。
