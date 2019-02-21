---
title: SpringBoot之Web进阶
date: 2017-06-30 01:00:00
tags: [SpringBoot]
categories:
- framework
- SpringBoot
---

项目代码：https://github.com/njZhuMin/SpringBootDemo
- - -

# 表单验证
假设我们现在建设一个特价图书的页面，规定特价区的图书价格不超过30.0元，那么当我们向这个页面添加图书信息时，就要校验价格是否合法。我们可以用`@Valid`注解来对表单参数进行验证。

首先我们看一下之前的写法：
```java
@RestController
public class BookController {
    @PostMapping("/books")
    public Book bookAdd(@RequestParam("name") String name,
                          @RequestParam("author") String author) {
        Book book = new Book();
        book.setName(name);
        book.setAuthor(author);
        return bookRepository.save(book);
    }
}
```
随着图书属性的增多（如价格，出版社，分类等等），我们就需要在传入的参数里不断使用`@RequestParam`来接收参数，显然是不现实的。

<!-- more -->

回顾一下SpringMVC中的参数绑定，我们可以看到在SpringMVC中就已经支持了对于POJO对象甚至更复杂的自定义复合对象参数绑定的支持。

> [https://zhum.in/blog/frame/SpringMVC/SpringMVC数据绑定/](https://zhum.in/blog/frame/SpringMVC/SpringMVC%E6%95%B0%E6%8D%AE%E7%BB%91%E5%AE%9A/)

因此这里我们可以放心的传入Book对象作为参数，Spring Boot也会自动的将它和我们的`@Entity`实体类进行参数绑定。我们现在实体类中注解需要进行校验的属性：
```java
@Entity
public class Book {

    public Book() { }

    @Id
    @GeneratedValue
    private Integer id;

    private String name;

    private String author;

    @DecimalMax(value = "30.0", message = "特价图书不超过30.0元")
    private float price;
}
```
可以看到这里我们使用了`@DecimalMax`注解来校验最大值不超过30.0，message属性则定义了当校验失败时默认返回的信息。

接下来我们在Controller中使用`@Valid`来标记需要属性校验的对象，并使用`BindingResult`对象获取校验返回的结果：
```java
@RestController
public class BookController {
    @PostMapping("/books")
    public Book bookAdd(@Valid Book book, BindingResult bindingResult) {
        if(bindingResult.hasErrors()) {
            System.out.println(bindingResult.getFieldError().getDefaultMessage());
            return null;
        }
        return bookRepository.save(book);
    }
}
```
这样我们就完成了图书对象的数据绑定，以及价格属性的合法性校验。

Spring Boot中提供的常见校验方式有：
| 约束注解名称 |                          约束注解说明                           |
| ------------ | --------------------------------------------------------------- |
| @null        | 验证对象是否为空                                                |
| @notnull     | 验证对象是否为非空                                              |
| @asserttrue  | 验证 boolean 对象是否为 true                                    |
| @assertfalse | 验证 boolean 对象是否为 false                                   |
| @min         | 验证 number 和 string 对象是否大等于指定的值                    |
| @max         | 验证 number 和 string 对象是否小等于指定的值                    |
| @decimalmin  | 验证 number 和 string 对象是否大等于指定的值，小数存在精度      |
| @decimalmax  | 验证 number 和 string 对象是否小等于指定的值，小数存在精度      |
| @size        | 验证对象（array,collection,map,string）长度是否在给定的范围之内 |
| @digits      | 验证 number 和 string 的构成是否合法                            |
| @past        | 验证 date 和 calendar 对象是否在当前时间之前                    |
| @future      | 验证 date 和 calendar 对象是否在当前时间之后                    |
| @pattern     | 验证 string 对象是否符合正则表达式的规则                        |
| @Email       | 验证邮箱                                                        |

下面举一些实际应用中的例子：
```java
@size(min=3, max=20, message="用户名长度只能在3-20之间")
@size(min=6, max=20, message="密码长度只能在6-20之间")
@pattern(regexp="[a-za-z0-9._%+-]+@[a-za-z0-9.-]+\\.[a-za-z]{2,4}", message="邮件格式错误")
@Length(min = 5, max = 20, message = "用户名长度必须位于5到20之间")  
@Email(message = "输入正确的邮箱格式不正确")  
@NotNull(message = "用户名称不能为空")
@Max(value = 100, message = "年龄不能大于100岁")
@Min(value= 18 ,message= "必须年满18岁！" )  
@AssertTrue(message = "bln4 must is true")
@AssertFalse(message = "blnf must is false")
@DecimalMax(value="100", message="decim最大值是100")
@DecimalMin(value="20", message="decim最小值是100")
@NotNull(message = "身份证号不能为空")
@Pattern(regexp="^(\\d{18,18}|\\d{15,15}|(\\d{17,17}[x|X]))$", message="身份证号格式错误")
```

# 使用AOP处理请求
AOP(Aspect Oriented Programming)，全称面向切面编程，是一种编程范式或设计思想，它的实现与语言无关。面向切面编程的核心思想就是将通用的逻辑从业务逻辑中分离出来。

我们通过日志拦截的例子来说明面向切面编程的编程：

{% asset_img log-aop.png log-aop %}

首先我们添加项目依赖：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```
然后我们定义一个Aspect来拦截`BookController`中的所有方法，并记录日志信息：
```java
public class HttpAspect {

    private final static Logger logger = LoggerFactory.getLogger(HttpAspect.class);
    @Pointcut("execution(public * com.sunnywr.controller.BookController.*(..))")
    public void log() { }

    /**
     * Http请求日志
     */
    @Before("log()")
    public void doBefore(JoinPoint joinPoint) {
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();
        // url
        logger.info("url={}", request.getRequestURL());
        // RequestMethod
        logger.info("method={}", request.getMethod());
        // ip
        logger.info("ip={}", request.getRemoteAddr());
        // 类方法
        logger.info("class_method={}", joinPoint.getSignature().getDeclaringTypeName()
                                + "." + joinPoint.getSignature().getName() + "()");
        // 参数
        logger.info("args={}", joinPoint.getArgs());
    }

    @After("log()")
    public void doAfter(JoinPoint joinPoint) {
        logger.info("Leaving {}()...", joinPoint.getSignature().getName());
    }

    @AfterReturning(returning = "object", pointcut = "log()")
    public void doAfterReturning(Object object) {
        logger.info("response={}", object.toString());
    }
}
```

# 统一异常处理
## 返回数据格式封装
我们先来回顾一下参数校验部分的代码：
```java
@RestController
public class BookController {
    @PostMapping("/books")
    public Book bookAdd(@Valid Book book, BindingResult bindingResult) {
        if(bindingResult.hasErrors()) {
            System.out.println(bindingResult.getFieldError().getDefaultMessage());
            return null;
        }
        return bookRepository.save(book);
    }
}
```
这里如果校验失败，则返回null并终止后续的数据库写入操作。这样看来逻辑是十分合理的，测试也没问题。但是我们加上了日志记录之后，就出现了新的问题：一旦校验失败，当日志记录返回结果时就会调用`null.toString()`方法输出，因此会出现空指针的异常。

我们希望接口能够返回统一的数据格式：
```js
// 校验失败
{
    "code": 1,
    "msg": "校验错误",
    "data": null
}
// 校验成功
{
    "code": 0,
    "msg": "成功",
    "data": {
        "id": 1,
        "name": "java入门",
        "author": "java",
        "price": 24.5
    }
}
```
定义一个Result对象来封装返回结果：
```java
public class Result<T> {
    private Integer errCode;
    private String msg;
    private T data;
    // ... getters and setters
}
```
再定义一个工具类来提炼重复代码逻辑：
```java
public class ResultUtil {

    public static Result successValidation(Object object) {
        Result result = new Result();
        result.setErrCode(0);
        result.setMsg("OK");
        result.setData(object);
        return result;
    }

    public static Result successValidation() {
        Result result = new Result();
        result.setErrCode(0);
        result.setMsg("OK");
        return result;
    }

    public static Result errorValidation(Integer errCode, String msg) {
        Result result = new Result();
        result.setErrCode(errCode);
        result.setMsg(msg);
        return result;
    }
}
```
这样我们就实现了返回数据格式的统一封装。

# 统一异常处理
## 自定义异常
假设我们现在需要判断图书的价格，来统计不同价格区间的图书信息，可能有这样的代码：
```java
@Service
public class BookService {
    public Integer getBookPrice(Integer id) {
        Book book = bookRepository.findOne(id);
        Float price = book.getPrice();
        if(price <= 20) {
            return code 1;
        } else if(price > 20 && price <= 30) {
            return code 2
        }
        // ...
        return code 0
    }
}
```
然后再在Controller层一个个判断：
```java
@RestController
public class BookController {
    @GetMapping("books/getPrice/{id}")
    public void getBookPrice(@PathVariable("id") Integer id) {
        Integer rCode = bookService.getBookPrice(id);
        if(rCode == 1)
            doSomething();
        else if(rCode == 2)
            doSomething();
        // ...
    }
}
```
这时就体现了统一异常处理的优势了。先在Service层条件逻辑处抛出异常：
```java
@Service
public class BookService {
    public void getBookPrice(Integer id) throws Exception {
        Book book = bookRepository.findOne(id);
        Float price = book.getPrice();
        if(price <= 20) {
            throw new Exception("0-20元");
        } else if(price > 20 && price <= 30) {
            throw new Exception("20-30元");
        }
    }
}
```
然后我们在Controller层继续向上抛出异常：
```java
@GetMapping("books/getBookPrice/{id}")
public void getBookPrice(@PathVariable("id") Integer id) throws Exception {
    bookService.getBookPrice(id);
}
```
然后我们定义一个ExceptionHandler来捕获并处理异常，其中`@ControllerAdvice`是SpringMVC中用来包装JSON异常的注解：
```java
@ControllerAdvice
public class MyExceptionHandler {

    @ExceptionHandler(value = Exception.class)
    @ResponseBody
    public Result exceptionHandler(Exception e) {
        return ResultUtil.errorValidation(20, e.getMessage());
    }
}
```
这样我们就可以统一捕获处理异常了。另一个问题是Exception默认只支持传入message一个参数，而我们希望可以给每一种情况自定义错误代码，因此我们自定义一个异常类。注意我们需要继承`RuntimeException`而不是`Exception`，因为Spring框架只对`RuntimeException`支持事务回滚。
```java
public class BookException extends  RuntimeException {

    private Integer errCode;

    public BookException(Integer errCode, String errMsg) {
        super(errMsg);
        this.errCode = errCode;
    }
    // ... getters and setters
}
```

使用日志来区分自定义异常和系统异常：
```java
@ControllerAdvice
public class MyExceptionHandler {

    private final static Logger logger = LoggerFactory.getLogger(MyExceptionHandler.class);

    @ExceptionHandler(value = Exception.class)
    @ResponseBody
    public Result exceptionHandler(Exception e) {
        if(e instanceof BookException) {
            BookException exception = (BookException) e;
            return ResultUtil.errorValidation(exception.getErrCode(), exception.getMessage());
        } else {
            logger.error("[system exception] {}", e);
            return ResultUtil.errorValidation(-1, "Undefined error.");
        }
    }
}
```

## 使用枚举类维护异常常量
定义枚举类型来维护异常常量：
```java
public enum ResultEnum {
    UNDEFINED_ERROR(-1, "Undefined error"),
    SUCCESS(0, "Success"),
    RANGE_0_20(20, "Price 0-20"),
    RANGE_20_30(30, "Price 20-30"),
    ;

    private Integer code;
    private String msg;

    ResultEnum(Integer code, String msg) {
        this.code = code;
        this.msg = msg;
    }

    // ... getters
}

```
修改异常构造方法：
```java
public class BookException extends  RuntimeException {

    private Integer errCode;

    public BookException(ResultEnum resultEnum) {
        super(resultEnum.getMsg());
        this.errCode = resultEnum.getCode();
    }

    // ... getters and setters
}
```

# 单元测试
一个合格的开发者，在开发完成后应当对代码进行单元测试。
首先我们对Service方法进行测试：
```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class BookServiceTest {

    @Autowired
    private BookService bookService;

    @Test
    public void findOneTest() {
        Book book = bookService.findOne(3);
        Assert.assertEquals(new Float(15), (Float)book.getPrice());
    }
}
```
然后我们对Controller进行测试。与Service方法测试不同，我们需要访问目标URL并测试返回的结果，因此这里我们使用`mockMvc`进行API测试：
```java
@RunWith(SpringRunner.class)
@SpringBootTest
@AutoConfigureMockMvc
public class BookControllerTest {

    @Autowired
    private MockMvc mvc;

    @Test
    public void bookList() throws Exception {
        mvc.perform(MockMvcRequestBuilders.get("/books"))
                .andExpect(MockMvcResultMatchers.status().isOk());
    }
}
```
其实当我们使用Maven打包项目时，Maven会自动帮我们执行所有的单元测试。要想跳过单元测试打包项目，只需要传入参数即可：
```bash
$ mvn clean package -D maven.test.skip=true
```
