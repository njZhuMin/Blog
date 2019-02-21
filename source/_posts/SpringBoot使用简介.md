---
title: SpringBoot使用简介
date: 2017-06-28 01:00:00
tags: [SpringBoot]
categories:
- framework
- SpringBoot
---

项目代码：https://github.com/njZhuMin/SpringBootDemo
- - -

# Spring Boot简介
多年以来，Spring IO平台饱受非议的一点就是大量的XML配置以及复杂的依赖管理。Spring Boot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。该框架使用了特定的方式来进行配置，开发人员不仅不再需要编写XML，而且在一些场景中甚至不需要编写繁琐的import语句。

Spring Boot的目标不在于为已解决的问题域提供新的解决方案，而是为平台带来另一种开发体验，从而简化对这些已有技术的使用。对于已经熟悉Spring生态系统的开发人员来说，Spring Boot是一个很理想的选择;对于采用Spring技术的新人来说，Spring Boot提供了一种更简洁的方式来使用这些技术。

<!-- more -->

在追求开发体验的提升方面，Spring Boot甚至整个Spring生态系统都使用到了Groovy编程语言。Spring Boot所提供的众多便捷功能，都是借助于Groovy强大的MetaObject协议、可插拔的AST转换过程以及内置的依赖解决方案引擎所实现的。在其核心的编译模型之中，Spring Boot使用Groovy来构建工程文件，所以它可以使用通用的导入和模板方法（如类的main方法）对类所生成的字节码进行装饰。

# Hello Spring Boot
我们先来写一个最简单的Spring Boot Demo程序。在IDEA中新建项目，选择`Spring Initializr`，使用默认的`https://start.spring.io/`接口来初始化项目配置，勾选`Web->web`并点击完成，IDEA会自动根据Maven配置文件下载所有依赖的包并构建项目结构。

项目自动配置完成后，可以看到在`src/main/java`下生成了一个默认的类`DemoApplication`：
```java
@SpringBootApplication
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```
我们自定义一个Controller类，拦截`/hello`并简单的返回一个字符串：
```java
@RestController
public class HelloController {

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String say() {
        return "Hello Spring Boot!";
    }
}
```
这样我们的第一个Spring Boot就构建好了。我们有三种方法来运行这个程序：

1. 在IDEA中添加`Spring Boot`运行配置，默认使用`resources/application.properties`属性启动项目，访问`http://localhost:8080/hello`即可看到Hello Spring Boot!

2. 在项目路径下运行`mvn spring-boot:run`来启动项目程序。需要配置好JDK、Maven环境变量，并且项目中必须包含`spring-boot-maven-plugin`组件依赖。

3. 在项目路径下运行`mvn install`，Maven会根据`pom.xml`中的`<packaging>jar</packaging>`配置编译构建`demo.jar`，然后我们只需要使用`java -jar demo.jar`，即可运行项目代码。

# 属性配置
Spring Boot中推荐使用YAML标记语言来代替默认的properties文件进行属性配置。结合YAML简洁的标记语法，我们可以轻松地向项目中注入配置。

## 单属性注入
使用`@Value`注解注入单属性值：
```yml
server:
    port: 8080
person:
    name: 隔壁老王
     age: 25
```
```java
@RestController
public class HelloController {
    @Value("${name}")
    private String name;
    @Value("${age}")
    private Integer age;

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String say() {
        return name + " " + age;
    }
}
```

## JavaBean注入
使用`@Component`与`@ConfigurationProperties`注入JavaBean对象：
```java
@Component
@ConfigurationProperties(prefix = "person")
public class DemoProperties {
    private String name;
    private Integer age;
    // ... getters and setters
}

@RestController
public class HelloController {
    @Autowired
    private PersonProperties personProperties;

    @RequestMapping(value = "/hello", method = RequestMethod.GET)
    public String say() {
        return personProperties.getName();
    }
}
```

## 多环境配置
Spring Boot还支持多个环境的配置。例如我们先定义顶层配置文件`application.yml`：
```yml
spring:
    profiles:
        active: deploy
```
然后分别定义开发环境与生产环境的配置文件`application-dev.yml`：
```yml
server:
    port: 8080
person:
  name: 隔壁老王
  age: 25
```
`application-deploy.yml`：
```yml
server:
    port: 8081
person:
  name: 隔壁老王
  age: 25
```
当前项目就会使用生产环境的配置运行，如果需要同时运行多个配置文件，可以先使用`mvn install`将项目编译为jar，然后传入`--spring.profiles.active=xxx`参数指定使用的配置文件：
```bash
$ java -jar demo.jar --spring.profiles.active=dev
$ java -jar demo.jar --spring.profiles.active=deploy
```

# Controller的使用
Spring Boot中关于Controller常用的注解有：
- @Controller： 声明当前类为Controller来处理http请求
- @RestController： 返回JSON格式，相当于@Controller和@ResponseBody的组合
- @RequestMapping： 配置URL映射与拦截

## 多URL映射
我们可以在`@RequestMapping`中传入URL的集合，来同时映射多个URL：
```java
@RestController
    public class HelloController {
    @RequestMapping(value = {"/hello", "/hi"}, method = RequestMethod.GET)
    public String say() {
        return personProperties.getName();
    }
}
```

## 多级URL映射
我们可以在类上加`@RequestMapping`注解来给整个Controller类添加URL映射，当我们访问时，需要将类上的URL与方法上的URL映射拼接起来`/hello/say`，即可被正确拦截响应：
```java
@RestController
@RequestMapping(value = {"/hello"}, method = RequestMethod.GET)
    public class HelloController {
    @RequestMapping(value = {"/say"}, method = RequestMethod.GET)
    public String say() {
        return personProperties.getName();
    }
}
```

## URL中的参数
Spring Boot中通过`@PathVariable`、`@RequestParam`和`@GetMapping`注解来处理URL中的参数问题。

例如使用`@PathVariable`来接收URL中的RESTFUL风格的参数：
```java
// http://localhost:8080/hello/say/123
@RestController
@RequestMapping(value = "/hello", method = RequestMethod.GET)
public class HelloController {
    @Autowired
    private PersonProperties personProperties;

    @RequestMapping(value = "/say/{id}", method = RequestMethod.GET)
    public String say(@PathVariable("id") Integer id) {
        return "id: " + id;
    }
}
```
使用`@RequestParam`接收传统类型的参数（如`?id=123`）：
```java
// http://localhost:8080/hello/say?id=123
@RestController
@RequestMapping(value = "/hello", method = RequestMethod.GET)
public class HelloController {
    @Autowired
    private PersonProperties personProperties;

    @RequestMapping(value = "/say", method = RequestMethod.GET)
    public String say(@RequestParam("id") Integer id) {
        return "id: " + id;
    }
}
```
这样的写法不允许缺少id参数，否则将会报错。我们可以进一步设置`RequestParam`注解的属性，让其可以处理不来id参数的URL，并设置id参数的默认值为0：
```java
// http://localhost:8080/hello/say?id=123
@RestController
@RequestMapping(value = "/hello", method = RequestMethod.GET)
public class HelloController {
    @Autowired
    private PersonProperties personProperties;

    @RequestMapping(value = "/say/{id}", method = RequestMethod.GET)
    public String say(@RequestParam(value = "id", required = false, defaultValue = "0") Integer id) {
        return "id: " + id;
    }
}
```
`@GetMapping`注解为我们提供了更加简化的URL Mapping定义方法：
```java
// @RequestMapping(value = "/say/{id}", method = RequestMethod.GET)
@GetMapping("/say/{id}")

// @RequestMapping(value = "/say/{id}", method = RequestMethod.POST)
@PostMapping("/say/{id}")
```

# 数据库操作
JPA全称Java Persistence API，它定义了一系列对象持久化的标准。目前实现这一规范的产品有Hibernate、TopLink等。而Spring-Data-Jpa简单地说就是Spring对Hibernate的整合。

我们具体来实现以下的RESTful API：

| 请求类型 | 请求路径  |        功能        |
| -------- | --------- | ------------------ |
| GET      | /books    | 获取图书列表       |
| POST     | /books    | 创建图书信息       |
| GET      | /books/id | 通过id查询图书信息 |
| PUT      | /books/id | 通过id更新图书信息 |
| DELETE   | /books/id | 通过id删除图书信息 |

## 项目依赖与配置
首先我们在项目中添加数据库支持的相关依赖：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.42</version>
</dependency>
```
然后我们配置数据库相关属性：
```yml
spring:
    datasource:
        driver-class-name: com.mysql.jdbc.Driver
        url: jdbc:mysql://127.0.0.1:3306/dbbook
        username: root
        password: 123456
    jpa:
        hibernate:
            ddl-auto: update
        show-sql: true
```

## 实体类与表映射
然后我们创建一个与表对应的实体类，使用`@Entity`标注这个类，同时这个类必须声明无参构造方法：
```java
@Entity
public class Book {
    public Book() { }

    @Id
    @GeneratedValue
    private Integer id;

    private String name;
    private String author;
}
```

## RESTful API的实现
在Spring Data Jpa中，实现数据库的操作十分简单，我们只需要先定义一个接口来继承`JpaRepository`：
```java
public interface BookRepository
        extends JpaRepository<Book, Integer> { }
```
然后在Service层中通过`@Autowired`注解注入并持有该实例，甚至不需要手写一行SQL语句，就可以完成数据库的操作。这里为了简化我们直接把DAO的逻辑展示在Controller中：
```java
@RestController
public class BookController {

    @Autowired
    private BookRepository bookRepository;

    /**
     * 查询所有图书列表
     * @return
     */
    @GetMapping("/books")
    public List<Book> bookList() {
        return bookRepository.findAll();
    }

    /**
     * 添加一本图书
     * @param name
     * @param author
     * @return
     */
    @PostMapping("/books")
    public Book bookAdd(@RequestParam("name") String name,
                          @RequestParam("author") String author) {
        Book book = new Book();
        book.setName(name);
        book.setAuthor(author);
        return bookRepository.save(book);
    }

    /**
     * 查询一本图书
     * @param id
     * @return
     */
    @GetMapping("/book/{id}")
    public Book bookQuery(@PathVariable("id") Integer id) {
        return bookRepository.findOne(id);
    }

    /**
     * 通过名称查询一本图书
     * @param name
     * @return
     */
    @GetMapping("/book/name/{name}")
    public List<Book> bookQueryByName(@PathVariable("name") String name) {
//        return bookRepository.findByName(name);
        return bookRepository.findByNameContaining(name);

    }

    /**
     * 更新一本图书
     * @param id
     * @return
     */
    @PutMapping("/book/{id}")
    public Book bookUpdate(@PathVariable("id") Integer id,
                           @RequestParam("name") String name,
                           @RequestParam("author") String author) {
        Book book = new Book();
        book.setId(id);
        book.setAuthor(author);
        book.setName(name);
        return bookRepository.save(book);
    }
}
```
这里的插入和更新操作对应的都是`repository.save()`方法，它会自动识别数据库中是否有这条记录，没有就执行插入，已经存在则执行更新操作。

> 在更新操作时，最好先根据ID查询出对象，再把需要更新的属性覆盖后，再做update，这样避免把不需要更新的字段置成NULL。

除了基本的增删改查数据库操作，Spring Data Jpa也支持复杂的数据库逻辑。我们同样只需要在`BookRepository`接口中声明查询方法，它就可以帮我们完成数据库操作：
```java
public interface BookRepository extends JpaRepository<Book, Integer> {
    // 通过名称查询
    List<Book> findByName(String name);

    List<Book> findByNameContaining(String name);
}
```
这里的方法命名规则要严格与数据库中的列名对应，如`findByName`；另外Jpa还提供了一些复杂查询方法的接口，如模糊查询`findByNameContaining`等。

# 数据库事务
与Spring中一致，Spring Boot中使用`@Transactional`来封装一个数据库事务：
```java
@Service
public class BookService {

    @Autowired
    private BookRepository bookRepository;

    @Transactional
    public void insertTwo() {
        Book book1 = new Book();
        book1.setAuthor("transaction");
        book1.setName("事务管理1");
        bookRepository.save(book1);

        Book book2 = new Book();
        book2.setAuthor("transaction");
        book2.setName("事务管理2");
        bookRepository.save(book2);
    }
}

@RestController
public class BookController {

    @Autowired
    private BookService bookService;

    /**
     * 数据库事务
     */
    @PostMapping("/books/two")
    public void bookInsertTwo() {
        bookService.insertTwo();
    }
}
```
