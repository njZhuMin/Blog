---
title: Log4j2的简单配置与使用
date: 2016-05-03 09:31:16
tags: [apache,log4j]
categories:
- framework
- logging
---

> [Log4j2系列文章](http://zhum.in/blog/categories/frame/log/)

- - -
# Hello World
使用前首先需要引入Log4j2的依赖：
```xml
<dependencies>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
        <version>2.6.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.6.1</version>
    </dependency>
</dependencies>
```
<!-- more -->

如果是在Web项目中使用，还需要引入`log4j-web`的依赖：
```xml
<dependencies>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-web</artifactId>
        <version>2.6.1</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

**[官方文档](http://logging.apache.org/log4j/2.x/manual/webapp.html)中有如下描述：**
> You must take particular care when using Log4j or any other logging framework within a Java EE web application. It's important for logging resources to be properly cleaned up (database connections closed, files closed, etc.) when the container shuts down or the web application is undeployed. Because of the nature of class loaders within web applications, Log4j resources cannot be cleaned up through normal means. Log4j must be "started" when the web application deploys and "shut down" when the web application undeploys. How this works varies depending on whether your application is a Servlet 3.0 or newer or Servlet 2.5 web application.
>
> In either case, you'll need to add the log4j-web module to your deployment as detailed in the Maven, Ivy, and Gradle Artifacts manual page.
>
> **To avoid problems the Log4j shutdown hook will automatically be disabled when the log4j-web jar is included.**

引入JUnit4依赖
```xml
<dependencies>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-web</artifactId>
        <version>2.6.1</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

# LogEvent分析
## 如何产生LogEvent

在调用Logger对象的info、error、trace等函数时，就会产生LogEvent。LogEvent跟LoggerConfig一样，也是有Level的。LogEvent的Level主要是用在Event传递时，判断在哪里停下。
```java
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
public class Test {
	private static Logger logger = LogManager.getLogger("HelloWorld");
	public static void main(String[] args){
    	Test.logger.info("hello,world");
		Test.logger.error("There is a error here");
	}
}
```
如代码中所示，这样就产生了两个LogEvent。

> 注意以下代码中的`logger1`和`logger2`是相同的对象。
```java
Logger logger1 = LogManager.getLogger("HelloWorld");
Logger logger2 = LogManager.getLogger("HelloWorld");
```

## LogEvent的传递是怎样的
给出一个示例程序如下：
```java
import org.apache.logging.log4j.Level;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class Log4j2Hello {
    private static Logger logger = LogManager.getLogger(Log4j2Hello.class.getName());
    public static void main(String[] args) {
        logger.trace("entry");  //trace级别的信息，单独列出来是希望你在某个方法或者程序逻辑开始的时候调用
        logger.error("Did it again!");   //error级别的信息，参数就是你输出的信息
        logger.info("我是info信息");    //info级别的信息
        logger.debug("我是debug信息");
        logger.warn("我是warn信息");
        logger.fatal("我是fatal信息");
        logger.log(Level.TRACE, "我是debug信息");   //这个就是制定Level类型的调用
        logger.trace("exit");   //和entry()对应的结束方法
    }
}
```
我们在IDE中运行一下这个程序，发现只有ERROR的语句输出了：
> 21:42:36.448 [main] ERROR Log4j2Hello - Did it again!
> 21:42:36.450 [main] FATAL Log4j2Hello - 我是fatal信息

那么INFO的语句呢？由于我们的工程中没有写入任何的配置文件，所以，application应该是使用了默认的LoggerConfig Level，默认的输出地是`SYSTEM_OUT`，即console，默认的级别是ERROR级别。
根据拦截器等级的限制关系，INFO级别的Event是无法被ERROR级别的LoggerConfig的filter接受的。所以，默认配置下INFO信息不会被输出。

# Log4j2的配置文件重定位

如果想要改变默认的配置，那么就需要configuration file。Log4j的配置是写在log4j.properties文件里面，但是 Log4j2=就可以写在XML和JSON文件里了 。

1. 放在classpath（src）下，以log4j2.xml命名：使用Log4j2的一般都约定俗成的写一个log4j2.xml放在src目录下使用。这一点没有争议

2. 将配置文件放到别处：在系统工程里面，将log4j2的配置文件放到src目录底下很不方便。如果能把工程中用到的所有配置文件都放在一个文件夹里面，当然就更整齐更好管理了。但是想要实现这一点，前提就是Log4j2的配置文件能重新定位到别处去，而不是放在classpath底下。

如果没有设置`log4j.configurationFile` system property的话， application将在classpath中按照如下查找顺序来找配置文件：
> log4j2-test.json 或 log4j2-test.jsn
> log4j2-test.xml
> log4j2.json 或 log4j2.jsn
> log4j2.xml

这就是为什么在src目录底下放log4j2.xml文件可以被识别的原因了。如果想将配置文件重命名并放到别处，就需要设置系统属性log4j.configurationFile，即在VM arguments中写入该属性的key和value：
`Dlog4j.configurationFile="/path/to/LogConfig.xml"`

## 新建`resource/log4j2.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="OFF">
    <appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
        </Console>
    </appenders>
    <loggers>
        <!--我们只让这个logger输出trace信息，其他的都是error级别-->
        <!--
            additivity开启的话，由于这个logger也是满足root的，所以会被打印两遍。
            不过root logger 的level是error，为什么Bar 里面的trace信息也被打印两遍呢
        -->
        <logger name="com.sunnywr.Log4j2Hello" level="TRACE" additivity="false">
            <appender-ref ref="Console"/>
        </logger>
        <root level="TRACE">
            <appender-ref ref="Console"/>
        </root>
    </loggers>
</configuration>
```

## 参数简单说明
1. 根节点configuration，然后有两个子节点：appenders和loggers（都是复数，意思就是可以定义很多个appender和logger了）（如果想详细的看一下这个xml的结构，可以去jar包下面去找xsd文件和dtd文件）

2. appenders：这个下面定义的是各个appender，就是输出了，有好多类别，这里也不多说（容易造成理解和解释上的压力，一开始也未必能听懂，等于白讲），先看这个例子，只有一个Console，这些节点可不是随便命名的，Console就是输出控制台的意思。然后就针对这个输出设置一些属性，这里设置了PatternLayout就是输出格式了，基本上是前面时间，线程，级别，logger名称，log信息等，差不多，可以自己去查他们的语法规则。

3. loggers下面会定义许多个logger，这些logger通过name进行区分，来对不同的logger配置不同的输出，方法是通过引用上面定义的logger，注意，appender-ref引用的值是上面每个appender的name，而不是节点名称。

# Log4j2的Name机制
> [官方文档](http://logging.apache.org/log4j/2.x/manual/architecture.html)

我们这里看到了配置文件里面是name很重要。没错，这个name可不能随便起（其实可以随便起）。这个机制意思很简单。就是类似于`Java package`一样，比如我们的一个包：com.example.base.log4j2。而且，可以发现我们前面生成Logger对象的时候，命名都是通过`Hello.class.getName();` 这样的方法，为什么要这样呢？ 很简单，因为有所谓的Logger 继承的问题。比如 如果你给com.example.base定义了一个logger，那么他也适用于com.example.base.lgo4j2这个logger。名称的继承是通过点（.）分隔的。然后你可以猜测上面loggers里面有一个子节点不是logger而是root，而且这个root没有name属性。这个root相当于根节点。你所有的logger都适用与这个logger，所以，即使你在很多类里面通过  类名.class.getName()  得到很多的logger，而且没有在配置文件的loggers下面做配置，他们也都能够输出，因为他们都继承了root的log配置。

我们上面的这个配置文件里面还定义了一个logger，他的名称是 com.example.base.log4j2.Hello，这个名称其实就是通过前面的Hello.class.getName(); 得到的，我们为了给他单独做配置，这里就生成对于这个类的logger，上面的配置基本的意思是只有com.example.base.log4j2.Hello 这个logger输出trace信息，也就是他的日志级别是trace，其他的logger则继承root的日志配置，日志级别是error，只能打印出ERROR及以上级别的日志。如果这里logger 的name属性改成com.example.base，则这个包下面的所有logger都会继承这个log配置（这里的包是log4j的logger name的“包”的含义，不是java的包，你非要给Hello生成一个名称为“myhello”的logger，他也就没法继承com.example.base这个配置了。

那有人就要问了，他不是也应该继承了root的配置了么，那么会不会输出两遍呢？我们在配置文件中给了解释，如果你设置了`additivity="false"`，就不会输出两遍。

举个稍微复杂的例子：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="error">
	<!--先定义所有的appender-->
    <appenders>
        <!--这个输出控制台的配置-->
        <Console name="Console" target="SYSTEM_OUT">
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <ThresholdFilter level="trace" onMatch="ACCEPT" onMismatch="DENY"/>
            <!--这个都知道是输出日志的格式-->
            <PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %class{36} %L %M - %msg%xEx%n"/>
        </Console>
        <!--文件会打印出所有信息，这个log每次运行程序会自动清空，由append属性决定，这个也挺有用的，适合临时测试用-->
        <File name="log" fileName="log/test.log" append="false">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %class{36} %L %M - %msg%xEx%n"/>
        </File>

        <!--这个会打印出所有的信息，每次大小超过size，则这size大小的日志会自动存入按年份-月份建立的文件夹下面并进行压缩，作为存档-->
        <RollingFile name="RollingFile" fileName="logs/app.log" filePattern="log/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz">
            <PatternLayout pattern="%d{yyyy-MM-dd 'at' HH:mm:ss z} %-5level %class{36} %L %M - %msg%xEx%n"/>
        	<SizeBasedTriggeringPolicy size="50MB"/>
        </RollingFile>
    </appenders>
    <!--然后定义logger，只有定义了logger并引入的appender，appender才会生效-->
    <loggers>
        <!--建立一个默认的root的logger-->
        <root level="trace">
            <appender-ref ref="RollingFile"/>
            <appender-ref ref="Console"/>
        </root>
    </loggers>
</configuration>
```

# 配置文件一些特征
## 扩展组件
1. ConsoleAppender
输出结果到System.out或是System.err。

2. FileAppender
输出结果到指定文件，同时可以指定输出数据的格式。append=“false”指定不追加到文件末尾

3. RollingFileAppender
自动追加日志信息到文件中，直至文件达到预定的大小，然后自动重新生成另外一个文件来记录之后的日志。

## 过滤标签
1. ThresholdFilter
用来过滤指定优先级的事件。

2. TimeFilter
设置start和end，来指定接收日志信息的时间区间。

## 常见日志需求
我们用日志一方面是为了记录程序运行的信息，在出错的时候排查之类的，有时候调试的时候也喜欢用日志。所以，日志如果记录的很乱的话，看起来也不方便。所以我可能有下面一些需求：

1. 我正在调试某个类，所以，我不想让其他的类或者包的日志输出，否则会很多内容，所以，你可以修改上面root的级别为最高（或者谨慎起见就用ERROR），然后，加一个针对该类的logger配置，比如第一个配置文件中的设置，把他的level设置trace或者debug之类的，然后我们给一个appender-ref是定义的File那个appender（共三个appender，还记得吗），这个appender的好处是有一个append为false的属性，这样，每次运行都会清空上次的日志，这样就不会因为一直在调试而增加这个文件的内容，查起来也方便，这个和输出到控制台就一个效果了。

2. 我已经基本上部署好程序了，然后我要长时间运行了。我需要记录下面几种日志，第一，控制台输出所有的error级别以上的信息。第二，我要有一个文件输出是所有的debug或者info以上的信息，类似做程序记录什么的。第三，我要单独为ERROR以上的信息输出到单独的文件，如果出了错，只查这个配置文件就好了，不会去处理太多的日志。怎么做呢，很简单：
 - 首先，在appenders下面加一个Console类型的appender，通过加一个ThresholdFilter设置level为error。（直接在配置文件的Console这个appender中修改）
 - 其次，增加一个File类型的appender（也可以是RollingFile或者其他文件输出类型），然后通过设置ThresholdFilter的level为error，设置成File好在，你的error日志应该不会那么多，不需要有多个error级别日志文件的存在，否则你的程序基本上可以重写了。

 - 这里可以添加一个appender，内容如下：
```xml
<File name="ERROR" fileName="logs/error.log">
	<ThresholdFilter level="error" onMatch="ACCEPT" onMismatch="DENY"/>
	<PatternLayout pattern="%d{yyyy.MM.dd 'at' HH:mm:ss z} %-5level %class{36} %L %M - %msg%xEx%n"/>
</File>
```
并在loggers中的某个logger(如root）中引用（root节点加入这一行作为子节点）：
```xml
 <appender-ref ref="ERROR" />
```
 - 然后，增加一个RollingFile的appender，设置基本上同上面的那个配置文件。

 - 最后，在logger中进行相应的配置。不过如果你的logger中也有日志级别的配置，如果级别都在error以上，你的appender里面也就不会输出error一下的信息了。
