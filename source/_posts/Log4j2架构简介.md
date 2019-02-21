---
title: Log4j2架构简介
date: 2016-05-02 15:31:16
tags: [apache,log4j]
categories:
- framework
- logging
---

> [Log4j2系列文章](http://zhum.in/blog/categories/frame/log/)

- - -
# 简介
## Log4j2介绍
Log4j2是Log4j的升级版，与之前的版本Log4j 1.x相比、有重大的改进，在修正了Logback固有的架构问题的同时，改进了许多Logback所具有的功能。

## Log4j2的特性及改进
- API分离： Log4j2将API与实现分离开来。开发人员现在可以很清楚地知道能够使用哪些没有兼容问题的类和方法，同时又允许通过自己实现来增强功能。

- 改进的性能： Log4j2的性能在某些关键领域比Log4j 1.x更快，而且大多数情况下与Logback相当。

- 多个API支持：Log4j2提供最棒的性能的同时，还支持SLF4J和公共日志记录API 。

- 自动配置加载：像Logback一样，一旦配置发生改变，Log4j2可以自动载入这些更改后的配置信息，又与Logback不同，配置发生改变时不会丢失任何日志事件。

<!-- more -->

- 高级过滤功能：与Logback类似， Log4j2可以支持基于上下文数据、标记，正则表达式以及日志事件中的其他组件的过滤 。Log4j2能够专门指定适用于所有的事件，无论这些事件在传入Loggers之前还是正在传给appenders。另外，过滤器还可以与Loggers关联起来。与Logback不同的是，Filter公共类可以用于任何情况。

- 插件架构：所有可以配置的组件都以Log4j插件的形式来定义 。同样地，不需要修改任何Log4j代码就可以创建新的Appender、Layout、Pattern Convert 等等。Log4j自动识别预定义的插件，如果在配置中引用到这些插件，Log4j就自动载入使用。

- 属性支持： 属性可以在配置文件中引用，也可以直接替代或传入潜在的组件，属性在这些组件中能够动态解析。属性可以是配置文件，系统属性，环境变量，线程上下文映射以及事件中的数据中定义的值。用户可以通过增加自己的Lookup插件来定制自己的属性。

## 更为先进的API（Modern API）
在这之前，程序员们以如下方式进行日志记录：
```java
if(logger.isDebugEnabled()) {
     logger.debug("Hi, " + u.getA() + “ “ + u.getB());
 }
```
许多人都会抱怨上述代码的可读性太差了。如果有人忘记写if语句，程序输出中会多出很多不必要的字符串。现在，Java虚拟机（JVM）也许对字符串的打印和输出进行了很多优化，但是难道我们仅仅依靠JVM优化来解决上述问题？log4j 2.0开发团队鉴于以上考虑对API进行了完善。现在你可以这样写代码：
```java
logger.debug("Hi, {} {}", u.getA(), u.getB());
```
和其它一些流行的日志框架一样， 新的API也支持变量参数的占位符功能。

log4j 2.0还支持其它一些很棒的功能，像Markers和flow tracing：
```java
private Logger logger = LogManager.getLogger(MyApp.class.getName());
private static final Marker QUERY_MARKER = MarkerManager.getMarker("SQL");
//...
public String doQuery(String table) {
	logger.entry(param);
	logger.debug(QUERY_MARKER, "SELECT * FROM {}", table);
	return logger.exit();
}
```
Markers可以帮助你很快地找到具体的日志项（Log Entries）。而在某个方法的开头和结尾调用Flow Traces中的一些方法，你可以在日志文件中看到很多新的跟踪层次的日志项，也就是说，你的程序工作流（Program Flow）被记录下来了。下面是Flow Traces的一些例子：
> 19:08:07.056 TRACE com.test.TestService 19 retrieveMessage - entry
> 19:08:07.060 TRACE com.test.TestService 46 getKey - entry

## 插件式的架构
Log4j 2.0支持插件式的架构 。你可以根据需要自行扩展log4j 2.0，这非常简单。首先，你要为你的扩展建立好命名空间，然后告诉log4j 2.0在哪能够找到它。
```xml
<configuration … packages="de.grobmeier.examples.log4j2.plugins">
```
根据上述配置，log4j 2将会在de.grobmeier.examples.log4j2.plugins包中找寻你的扩展插件。如果你建立了多个命名空间，没关系，用逗号分隔就可以了。

下面是一个简单的扩展插件：
```java
@Plugin(name = "Sandbox", type = "Core", elementType = "appender")
public class SandboxAppender extends AppenderBase {
	private SandboxAppender(String name, Filter filter) {
		super(name, filter, null);
	}

     public void append(LogEvent event) {
         System.out.println(event.getMessage().getFormattedMessage());
     }

	@PluginFactory
	public static SandboxAppender createAppender(
		@PluginAttr("name") String name, @PluginElement("filters") Filter filter) {
		return new SandboxAppender(name, filter);
     }
}
```
上面标有 @PluginFactory注解的方法是一个工厂，它的两个参数直接从配置文件读取。我用@PluginAttr和@PluginElement进行了实现。

剩下的就非常简单了。由于我写的是一个Appender，因此得继承AppenderBase这个类。该类必须实现append()方法，从而进行实际的逻辑处理。除了Appender，你甚至可以实现自己的Logger和Filter。

强大的配置功能（Powerful Configuration）log4j2的配置变得非常简单。如果你习惯了之前的配置方式，也不用担心，你只要花很少的时间就可以从之前的方式转换到新的方式。请看下面的配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration status="OFF">
	<appenders>
		<Console name="Console" target="SYSTEM_OUT">
			<PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n"/>
		</Console>
	</appenders>
	<loggers>
		<logger name="com.foo.Bar" level="trace" additivity="false">
			<appender-ref ref="Console"/>
		</logger>
		<root level="error">
             <appender-ref ref="Console"/>
		</root>
	</loggers>
</configuration>
```
上面说的只是一部分改进， 你还可以自动重新加载配置文件 ：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration monitorInterval="30">
 ...
</configuration>
```
监控的时间间隔单位为秒，最小值是5。这意味着，log4j 2在配置改变的情况下可以重新配置日志记录行为。如果值设置为0或负数，log4j 2不会对配置变更进行监测。 最为称道的一点是：不像其它日志框架，log4j 2.0在重新配置的时候不会丢失之前的日志记录。

还有一个非常不错的改进，那就是：同XML相比，如果你更加喜欢JSON，你可以自由地进行基于JSON的配置了：
```json
{
	 "configuration": {
     	"appenders": {
        	"Console": {
             	"name": "STDOUT",
             	"PatternLayout": {
                 	"pattern": "%m%n"
             	}
         	}
     	},
     	"loggers": {
         	"logger": {
             	"name": "EventLogger",
             	"level": "info",
             	"additivity": "false",
             	"appender-ref": {
                 	"ref": "Routing"
             	}
         	},
         	"root": {
             	"level": "error",
             	"appender-ref": {
                 	"ref": "STDOUT"
             	}
         	}
     	}
 	}
}
```
Java5 并发性（Concurrency） 有一段文档是这样描述的
> Log4j 2利用Javab5 中的并发特性支持，尽可能地执行最低层次的加锁...

Apache log4j 2.0解决了许多在log4j 1.x中仍然存留的死锁问题。如果你的程序仍然饱受内存泄漏的折磨，请毫不犹豫地试一下log4j 2.0。

# 日志级别
我们现在要调用logger的方法，不过在这个Logger对象中，有很多方法，所以要先了解log4j的日志级别，log4j规定了默认的几个级别：`trace < debug < info < warn < error < fatal`等。这里要说明一下：
1. 级别之间是包含的关系，意思是如果你设置日志级别是trace，则大于等于这个级别的日志都会输出。

2. 基本上默认的级别没多大区别，就是一个默认的设定。你可以通过它的API自己定义级别。你也可以随意调用这些方法，不过你要在配置文件里面好好处理了，否则就起不到日志的作用了，而且也不易读，相当于一个规范，你要完全定义一套也可以，不用没多大必要。

3. 这不同的级别的含义大家都很容易理解，这里就简单介绍一下：
- trace： 追踪，就是程序推进一下，你就可以写个trace输出，所以trace应该会特别多，不过没关系，我们可以设置最低日志级别不让他输出。

- debug： 调试么，我一般就只用这个作为最低级别，trace压根不用。实在没办法就用Eclipse或者IDEA的debug功能就好了么。

- info： 输出一下你感兴趣的或者重要的信息，这个用的最多了。

- warn： 有些信息不是错误信息，但是也要给程序员的一些提示，类似于Eclipse中代码的验证不是有error和warn（不算错误但是也请注意，比如以下depressed的方法）。

- error： 错误信息。用的也比较多。

- fatal： 级别比较高了。重大错误，这种级别你可以直接停止程序了，是不应该出现的错误么！不用那么紧张，其实就是一个程度的问题。

# Log4j2类图
{% asset_img log4j2.png log4j2 %}

通过类图可用看到：

每一个log上下文对应一个configuration，configuration中详细描述了log系统的各个LoggerConfig、Appender（输出目的地）、EventLog过滤器等。每一个Logger又与一个LoggerConfig相关联。另外，可以看到Filter的种类很多，有聚合在Configuration中的filter、有聚合在LoggerConfig中的filter也有聚合在Appender中的filter。不同的filter在过滤LogEvent时的行为和判断依据是不同的，具体可参加本文后面给出的文档。

应用程序通过调用log4j2的API并传入一个特定的名称来向LogManager请求一个Logger实例。LogManager会定位到适当的 LoggerContext 然后通过它获得一个Logger。如果LogManager不得不新建一个Logger，那么这个被新建的Logger将与LoggerConfig相关联，这个LoggerConfig的名称中包含如下信息中的一种：
> 1. 与Logger名称相同的
> 2. 父logger的名称
> 3. root

当一个LoggerConfig的名称与一个Logger的名称可以完全匹配时，Logger将会选择这个LoggerConfig作为自己的配置。如果不能完全匹配，那么Logger将按照最长匹配串来选择自己所对应的LoggerConfig。LoggerConfig对象是根据配置文件来创建的。LoggerConfig会与Appenders相关联，Appenders用来决定一个log request将被打印到那个目的地中，可选的打印目的地很多，如console、文件、远程socket server等。LogEvent是由Appenders来实际传递到最终输出目的地的，而在EvenLog到达最终被处理之前，还需要经过若干filter的过滤，用来判断该EventLog应该在何处被转发、何处被驳回、何处被执行。

# 模块间的引用关系
## Logger间的层次关系

相比于纯粹的System.out.println方式，使用logging API的最首要以及最重要的优势是可以在禁用一些log语句块的同时允许其他的语句块的输出。这一能力建立在一种假设之上，即所有在应用中可能出现的logging语句可以按照开发者定义的标准分成不同的类型。

在 Log4j 1.x版本时，Logger的层次是靠Logger类之间的关系来维护的。但在Log4j2中， Logger的层次则是靠LoggerConfig对象之间的关系来维护的。

Logger和LoggerConfig均是有名称的实体。Logger的命名是大小写敏感的，并且服从如下的分层命名规则。（与java包的层级关系类似）。例如：com.foo是com.foo.Bar的父级；java是java.util的父级，是java.util.vector的祖先。

root LoggerConfig位于LoggerConfig层级关系的最顶层。它将永远存在与任何LoggerConfig层次中。任何一个希望与root LoggerConfig相关联的Logger可以通过如下方式获得：
```java
Logger logger = LogManager.getLogger(LogManager.ROOT_LOGGER_NAME);
```

其他的Logger实例可以调用LogManager.getLogger 静态方法并传入想要得到的Logger的名称来获得。

## LoggerContext

LoggerContext在Logging System中扮演了锚点的角色。根据情况的不同，一个应用可能同时存在于多个有效的LoggerContext中。在同一LoggerContext下，log system是互通的。如：Standalone Application、Web Applications、Java EE Applications、"Shared" Web Applications 和REST Service Containers，就是不同广度范围的log上下文环境。

## Configuration

每一个LoggerContext都有一个有效的Configuration。Configuration包含了所有的Appenders、上下文范围内的过滤器、LoggerConfigs以及StrSubstitutor.的引用。在重配置期间，新与旧的Configuration将同时存在。当所有的Logger对象都被重定向到新的Configuration对象后，旧的Configuration对象将被停用和丢弃。

## Logger

如前面所述， Loggers 是通过调用LogManager.getLogger方法获得的。Logger对象本身并不实行任何实际的动作。它只是拥有一个name 以及与一个LoggerConfig相关联。它继承了AbstractLogger类并实现了所需的方法。当Configuration改变时，Logger将会与另外的LoggerConfig相关联，从而改变这个Logger的行为。

- 获得Logger：

> 使用相同的名称参数来调用getLogger方法将获得来自同一个Logger的引用。如：
```java
//x和y指向的是同一个Logger对象
Logger x = Logger.getLogger("wombat");
Logger y = Logger.getLogger("wombat");
```

log4j环境的配置是在应用的启动阶段完成的。优先进行的方式是通过读取配置文件来完成。

log4j使采用类名（包括完整路径）来定义Logger 名变得很容易。这是一个很有用且很直接的Logger命名方式。使用这种方式命名可以很容易的定位这个log message产生的类的位置。当然，log4j也支持任意string的命名方式以满足开发者的需要。不过，使用类名来定义Logger名仍然是最为推崇的一种Logger命名方式。

## LoggerConfig

当Logger在configuration中被描述时，LoggerConfig对象将被创建。LoggerConfig包含了一组过滤器。LogEvent在被传往Appender之前将先经过这些过滤器。过滤器中包含了一组Appender的引用。Appender则是用来处理这些LogEvent的。

## Log层次：

每一个LoggerConfig会被指定一个Log层次。可用的Log层次包括TRACE, DEBUG,INFO, WARN, ERROR 以及FATAL。需要注意的是，在log4j2中，Log的层次是一个Enum型变量，是不能继承或者修改的。如果希望获得跟多的分割粒度，可用考虑使用Markers来替代。

在Log4j 1.x 和Logback 中都有“层次继承”这么个概念。但是在log4j2中，由于Logger和LoggerConfig是两种不同的对象，因此“层次继承”的概念实现起来跟Log4j 1.x 和Logback不同。具体情况下面的五个例子。

例子一：

| Logger Name | Assigned LoggerConfig  | Level  |
| ----------- | ------------ | ----- |
| root	      | root		 | DEBUG |
| X			  | root		 | DEBUG |
| X.Y	      | root		 | DEBUG |
| X.Y.Z	      | root		 | DEBUG |

可用看到，应用中的LoggerConfig只有root这一种。因此，对于所有的Logger而言，都只能与该LoggerConfig相关联而没有别的选择。

例子二：

| Logger Name | Assigned LoggerConfig  | Level  |
| ----------- | ------------ | ----- |
| root	      | root		 | DEBUG |
| X			  | X			 | ERROR |
| X.Y	      | X.Y		 	 | INFO  |
| X.Y.Z	      | X.Y.Z		 | WARN  |

在例子二中可以看到，有5种不同的LoggerConfig存在于应用中，而每一个Logger都被与最匹配的LoggerConfig相关联着，并且拥有不同的Log Level。

例子三：

| Logger Name | Assigned LoggerConfig  | Level  |
| ----------- | ------------ | ----- |
| root	      | root		 | DEBUG |
| X			  | X			 | ERROR |
| X.Y	      | X		 	 | ERROR |
| X.Y.Z	      | X.Y.Z		 | WARN  |


可以看到Logger root、X、X.Y.Z都找到了与各种名称相同的LoggerConfig。而LoggerX.Y没有与其名称相完全相同的LoggerConfig。怎么办呢？它最后选择了X作为它的LoggerConfig，因为X LoggerConfig拥有与其最长的匹配度。

例子四：

| Logger Name | Assigned LoggerConfig  | Level  |
| ----------- | ------------ | ----- |
| root	      | root		 | DEBUG |
| X			  | X			 | ERROR |
| X.Y	      | X		 	 | ERROR |
| X.Y.Z	      | X			 | ERROR |


可以看到，现在应用中有两个配置好的LoggerConfig：root和X。而Logger有四个：root、X、X.Y、X.Y.Z。其中，root和X都能找到完全匹配的LoggerConfig，而X.Y和X.Y.Z则没有完全匹配的LoggerConfig，那么它们将选择哪个LoggerConfig作为自己的LoggerConfig呢？由图上可知，它们都选择了X而不是root作为自己的LoggerConfig，因为在名称上，X拥有最长的匹配度。

例子五

| Logger Name | Assigned LoggerConfig  | Level  |
| ----------- | ------------ | ----- |
| root	      | root		 | DEBUG |
| X			  | X			 | ERROR |
| X.Y	      | X.Y		 	 | INFO  |
| X.Y.Z	      | X			 | ERROR |


可以看到，现在应用中有三个配置好的LoggerConfig，分别为：root、X、X.Y。同时，有四个Logger，分别为：root、X、X.Y以及X.YZ。其中，名字能完全匹配的是root、X、X.Y。那么剩下的X.YZ应该匹配X还是匹配X.Y呢？答案是X。因为匹配是按照标记点（即“.”）来进行的，只有两个标记点之间的字串完全匹配才算，否则将取上一段完全匹配的字串的长度作为最终匹配长度。

## Filter

与防火墙过滤的规则相似，log4j2的过滤器也将返回三类状态：Accept（接受）, Deny（拒绝） 或 Neutral（中立）。其中，Accept意味着不用再调用其他过滤器了，这个LogEvent将被执行；Deny意味着马上忽略这个event，并将此event的控制权交还给过滤器的调用者；Neutral则意味着这个event应该传递给别的过滤器，如果再没有别的过滤器可以传递了，那么就由现在这个过滤器来处理。

## Appender

由logger的不同来决定一个logging request是被禁用还是启用只是log4j2的情景之一。log4j2还允许将logging request中log信息打印到不同的目的地中。在log4j2的世界里，不同的输出位置被称为Appender。目前，Appender可以是console、文件、远程socket服务器、Apache Flume、JMS以及远程 UNIX 系统日志守护进程。一个Logger可以绑定多个不同的Appender。

可以调用当前Configuration的addLoggerAppender函数来为一个Logger增加。如果不存在一个与Logger名称相对应的LoggerConfig，那么相应的LoggerConfig将被创建，并且新增加的Appender将被添加到此新建的LoggerConfig中。尔后，所有的Loggers将会被通知更新自己的LoggerConfig引用（PS：一个Logger的LoggerConfig引用是根据名称的匹配长度来决定的，当新的LoggerConfig被创建后，会引发一轮配对洗牌）。

在某一个Logger中被启用的logging request将被转发到该Logger相关联的的所有Appenders上，并且还会被转发到LoggerConfig的父级的Appenders上。

这样会产生一连串的遗传效应。例如，对LoggerConfig B来说，它的父级为A，A的父级为root。如果在root中定义了一个Appender为console，那么所有启用了的logging request都会在console中打印出来。另外，如果LoggerConfig A定义了一个文件作为Appender，那么使用LoggerConfig A和LoggerConfig B的logger 的logging request都会在该文件中打印，并且同时在console中打印。

如果想避免这种遗传效应的话，可以在configuration文件中做如下设置：
`additivity="false"`

这样，就可以关闭Appender的遗传效应了。具体解释见：

> **Appender Additivity**
> The output of a log statement of Logger L will go to all the Appenders in the LoggerConfig associated with L and the ancestors of that LoggerConfig. This is the meaning of the term "appender additivity".
> However, if an ancestor of the LoggerConfig associated with Logger L, say P, has the additivity ﬂag set to `false`, then L's output will be directed to all the appenders in L's LoggerConfig and its ancestors up to and including P but not the Appenders in any of the ancestors of P.
> Loggers have their additivity ﬂag set to `true` by default.

## Layout

通常，用户不止希望能定义log输出的位置，还希望可以定义输出的格式。这就可以通过将Appender与一个layout相关联来实现。Log4j中定义了一种类似C语言printf函数的打印格式，如"%r [%t] %-5p %c - %m%n" 格式在真实环境下会打印类似如下的信息：

`176 [main] INFO  org.foo.Bar - Located nearest gas station.`

其中，各个字段的含义分别是：
> %r 指的是程序运行至输出这句话所经过的时间（以毫秒为单位）；
> %t 指的是发起这一log request的线程；
> %c 指的是log的level；
> %m 指的是log request语句携带的message。
> %n 为换行符。