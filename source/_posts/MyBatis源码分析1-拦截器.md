---
title: MyBatis源码分析1-拦截器
date: 2016-05-28 09:31:16
tags: [mybatis,interceptor]
categories:
- framework
- MyBatis
---

> MyBatis系列文章：
> https://zhum.in/blog/categories/frame/MyBatis/

> 项目代码：
> https://github.com/njZhuMin/AutoReplyRobot

- - -

# 面向接口式编程
首先来看`MessageDao`中对`sqlSession.selectList`方法的调用：
```java
messageList = sqlSession.selectList("Message.queryMessageList", message);
```
跟踪一下这个函数的原型：
```java
public abstract <E> java.util.List<E> selectList(String statement, Object parameter)
```
可以看到这里可能存在4个风险：
1. 对传入的`"Message.queryMessageList"`中的namespace `Message`有效性的检查
2. parameter是Object，因此传入任何对象都可以，但是如何在MyBatis使用中做检查
3. 返回值类型为List，但是没有泛型约束，因此任何一个继承List的类都可以接收返回值，那么如何约束
4. `queryMessageList`与xml文件中配置的sql语句id是否对应

<!-- more -->

因此MyBatis提供面向接口式编程：
1. 声明接口，定义与xml中的id名称相同的方法，返回值为需要的List泛型，参数为需要传入的类型参数：
```java
package com.sunnywr.dao;
public interface MessageImpl {
    public List<Message> queryMessageList(Message message);
}
```
2. 在xml文件中配置namespace为接口类的reference：
```xml
<mapper namespace="com.sunnywr.dao.MessageImpl">
```
3. 使用`sqlSession`对象的`getMapper`方法获取接口，并调用与id同名方法：
```java
MessageImpl imessage = sqlSession.getMapper(MessageImpl.class);
messageList = imessage.queryMessageList(message);
```

至此通过面向接口的编程方法消除了原来可能存在的风险。

> 更进一步，当MyBatis与Spring整合后，MyBatis根配置文件中的数据源将托管给spring来管理，DB层将消失。

> 同时`sqlsession`对象将会托管给spring，组织对象的代码移交给service层，接口式编程将统一由spring来实现，整个dao层将会由之前的接口来替换。

# 几个问题
## imessage.queryMessageList()方法
`imessage.queryMessageList()`方法为什么可以实现查询？因为它实现了对MyBatis的的动态代理。过程如下：

1. 定义MapperProxy类实现InvocationHandler接口，并实现`invoke()`方法

2. 通过`MapperProxy.newProxyInstance(ClassLoader, infce, MapperProxyObject)`方法创建一个代理实例，即与`sqlSession.getMapper()`方法获取实例是一样的

3. `IMessage imessage = Proxy.newProxyInstance()`获取实例后，调用`imessage.queryMessageList()`实际上即是触发了`MapperProxy.invoke()`方法

## MapperProxy.invoke()
`MapperProxy.invoke()`方法如何准确调用`sqlSession.selectList()`方法？

1. 我们在获取`sqlSession`对象之前，首先完成加载MyBatis配置文件。通过MyBatis中的`Configuration`类解读配置文件

2. 由面向接口式编程，我们容易想到接口与配置文件间可能存在的对应关系：`接口全名.方法名 == namespace.id`

3. 通过`invoke()`方法中对`sqlSession.selectList(namespace.id, parameter)`的实现完成调用


> 通过上面的分析我们可以证明：

> `imessage.queryMessageList(parameter) == sqlSession.selectList(namespace.id, parameter)`

## 动态代理的泛型转换
在动态代理中，`Proxy.newProxyInstance()`方法返回的是Object类型值，为什么可以使用`IMessage imessage = Proxy.newProxyInstance()`呢？

这是因为MyBatis通过传入的接口的class自动判断并完成了泛型转换。

# 简单分页的实现
## 实现思路
- 实现一个实体对象Page，封装分页所需要的参数

```java
// 分页对应的实体类
public class Page {

    //总条数
    private int totalNumber;

    // 当前第几页
    private int currentPage;

    // 总页数
    private int totalPage;

    // 每页显示条数
    private int pageNumber = 5;

    // 数据库中limit的参数，从第几条开始取
    private int dbIndex;

    // 数据库中limit的参数，一共取多少条
    private int dbNumber;

    // 根据当前对象中属性值计算并设置相关属性值
    public void count() {
        // 计算总页数
        int totalPageTemp = this.totalNumber / this.pageNumber;
        int plus = (this.totalNumber % this.pageNumber) == 0 ? 0 : 1;
        totalPageTemp = totalPageTemp + plus;
        if(totalPageTemp <= 0) {
            totalPageTemp = 1;
        }
        this.totalPage = totalPageTemp;

        // 设置当前页数
        // 总页数小于当前页数，应将当前页数设置为总页数
        if(this.totalPage < this.currentPage) {
            this.currentPage = this.totalPage;
        }
        // 当前页数小于1设置为1
        if(this.currentPage < 1) {
            this.currentPage = 1;
        }

        // 设置limit的参数
        this.dbIndex = (this.currentPage - 1) * this.pageNumber;
        this.dbNumber = this.pageNumber;
    }

	//... getters and setters
}
```

- xml配置

```xml
<select id="queryMessageList" parameterType="java.util.Map" resultMap="MessageResult">
    SELECT <include refid="columns"/> FROM message
    <where>
        <if test="message.command != null and !&quot;&quot;.equals(message.command.trim())">
            AND COMMAND=#{message.command}
        </if>
        <if test="message.description != null and !&quot;&quot;.equals(message.description.trim())">
             AND DESCRIPTION LIKE '%' #{message.description} '%'
        </if>
    </where>
    ORDER BY ID LIMIT #{page.dbIndex},#{page.dbNumber}
</select>

<select id="count"  parameterType="com.sunnywr.bean.Message" resultType="int">
    SELECT COUNT(*) FROM message
    <where>
        <if test="command != null and !&quot;&quot;.equals(command.trim())">
            AND COMMAND=#{command}
        </if>
        <if test="description != null and !&quot;&quot;.equals(description.trim())">
            AND DESCRIPTION LIKE '%' #{description} '%'
        </if>
    </where>
</select>
```

- 前端JSP传递参数

```xml
<div class='page fix'>
	共 <b>${page.totalNumber}</b> 条
	<c:if test="${page.currentPage != 1}">
		<a href="javascript:changeCurrentPage('1')" class='first'>首页</a>
		<a href="javascript:changeCurrentPage('${page.currentPage-1}')" class='pre'>上一页</a>
	</c:if>
	当前第<span>${page.currentPage}/${page.totalPage}</span>页
	<c:if test="${page.currentPage != page.totalPage}">
		<a href="javascript:changeCurrentPage('${page.currentPage+1}')" class='next'>下一页</a>
		<a href="javascript:changeCurrentPage('${page.totalPage}')" class='last'>末页</a>
	</c:if>
	跳至&nbsp;<input id="currentPageText" type='text' value='${page.currentPage}' class='allInput w28' />&nbsp;页&nbsp;
	<a href="javascript:changeCurrentPage($('#currentPageText').val())" class='go'>GO</a>
</div>
```

- Servlet接收参数

```java
protected void doGet(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {
    // 设置编码
    req.setCharacterEncoding("UTF-8");
    // 接受页面的值
    String command = req.getParameter("command");
    String description = req.getParameter("description");
    String currentPage = req.getParameter("currentPage");
    // 创建分页对象
    Page page = new Page();
    // 前端传递参数验证
    Pattern pattern = Pattern.compile("[0-9]{1,9}");
    if(currentPage == null ||  !pattern.matcher(currentPage).matches()) {
        page.setCurrentPage(1);
    } else {
        page.setCurrentPage(Integer.valueOf(currentPage));
    }
    QueryService listService = new QueryService();
    // 查询消息列表并传给页面
    req.setAttribute("messageList", listService.queryMessageListByPage(command, description,page));
    // 向页面传值
    req.setAttribute("command", command);
    req.setAttribute("description", description);
    req.setAttribute("page", page);
    // 向页面跳转
    req.getRequestDispatcher("/WEB-INF/jsp/back/list.jsp").forward(req, resp);
}
```

- Service层处理参数

```java
public List<Message> queryMessageList(String command, String description, Page page) {
    // 组织消息对象
    Message message = new Message();
    message.setCommand(command);
    message.setDescription(description);
    MessageDao messageDao = new MessageDao();
    // 根据条件查询条数
    int totalNumber = messageDao.count(message);
    // 组织分页查询参数
    page.setTotalNumber(totalNumber);
    Map<String,Object> parameter = new HashMap<String, Object>();
    parameter.put("message", message);
    parameter.put("page", page);
    // 分页查询并返回结果
    return messageDao.queryMessageList(parameter);
}
```

- DAO层

```java
public List<Message> queryMessageList(Map<String,Object> parameter) {
    DBAccess dbAccess = new DBAccess();
    List<Message> messageList = new ArrayList<Message>();
    SqlSession sqlSession = null;
    try {
        sqlSession = dbAccess.getSqlSession();
        // 通过sqlSession执行SQL语句
        MessageImpl imessage = sqlSession.getMapper(MessageImpl.class);
        messageList = imessage.queryMessageList(parameter);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if(sqlSession != null) {
            sqlSession.close();
        }
    }
    return messageList;
}

/**
 * 根据查询条件查询消息列表的条数
 */
public int count(Message message) {
    DBAccess dbAccess = new DBAccess();
    SqlSession sqlSession = null;
    int result = 0;
    try {
        sqlSession = dbAccess.getSqlSession();
        // 通过sqlSession执行SQL语句
        MessageImpl imessage = sqlSession.getMapper(MessageImpl.class);
        result = imessage.count(message);
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        if(sqlSession != null) {
            sqlSession.close();
        }
    }
    return result;
}
```

## 几个注意点
1. MySQL中使用`LIMIT index,num`实现分页，最好使用`ORDER BY`定义排序，否则不能保证每次查询的排序相同

2. 前端的JavaScript校验不能保证安全，后端仍需做合法性校验

3. 分页功能中对当前页面的一些逻辑细节

## 问题提出
真实项目中，需要分页的列表页面肯定不止一个。按照上述方法需要开发无数次，显然是不显现实的。而且整个流程中存在大量重复代码。

# MyBatis拦截器
之前我们已经有利用log4j输出MyBatis执行的SQL语句的经历，因此考虑是否可以在MyBatis执行SQL语句之前取到SQL语句，加以改造再进行执行，即在MyBatis正常执行流程的基础上实现分页功能。

## 拦截功能的一些问题
1. 拦截什么样的对象

2. 拦截对象的什么行为

3. 什么时候拦截：在PreparedStatement执行之前拦截

## 拦截对象
首先我们新建一个拦截器类，实现MyBatis提供的`Interceptor`接口。实现这个接口需要实现`intercept`、`plugin`和`setProperties`三个方法。

通过动态代理的结构，我们找到MyBatis中的`StatementHandler`，这个接口定义了对SQL语句处理的事务逻辑。而其中的`prepare`函数则是我们比较关心的。
```java
public interface StatementHandler {
Statement prepare(Connection connection, Integer transactionTimeout)
      throws SQLException;
}
```
这个函数具体在`BaseStatementHandler`类中实现：
```java
public Statement prepare(Connection connection, Integer transactionTimeout) throws SQLException {
    ErrorContext.instance().sql(boundSql.getSql());
    Statement statement = null;
    try {
      statement = instantiateStatement(connection);
      setStatementTimeout(statement, transactionTimeout);
      setFetchSize(statement);
      return statement;
	} // catch...
}
```
可以看到这里使用`instantiateStatement`函数对statement做初始化。继续跟踪`instantiateStatement`函数，它是定义在`BaseStatementHandler`中的抽象方法。继续跟踪到它在`PreparedStatementHandler`中的实现：
```java
 protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
    if (mappedStatement.getKeyGenerator() instanceof Jdbc3KeyGenerator) {
      String[] keyColumnNames = mappedStatement.getKeyColumns();
      if (keyColumnNames == null) {
        return connection.prepareStatement(sql, PreparedStatement.RETURN_GENERATED_KEYS);
      } else {
        return connection.prepareStatement(sql, keyColumnNames);
      }
	  //...
    }
}
```
这里我们看到了类似JDBC中的实现：`return connection.prepareStatement(sql, keyColumnNames);`

这样我们就确定了需要拦截的目标，即`StatementHandler`类中的`prepare`方法。在MyBatis中提供了拦截器注解：
```java
@Intercepts({@Signature(type = StatementHandler.class,
        method = "prepare", args = {Connection.class, Integer.class})})
```
这里type定义了拦截的类，method定义拦截方法的名称。而根据反射原理，方法名和参数才唯一确定方法，所以args中我们传入prepare的参数列表`Connection.class, Integer.class`。

## 拦截行为
我们继续看`PreparedStatementHandler`类中的`instantiateStatement`方法，它其实已经的到了从配置文件传入的sql语句：
```java
 protected Statement instantiateStatement(Connection connection) throws SQLException {
    String sql = boundSql.getSql();
	//...
}
```
因此我们需要想办法得到这个sql语句并加以改造。

回到拦截器，我们首先实现`plugin`方法：
```java
public Object plugin(Object target) {
    return Plugin.wrap(target, this);
}
```
这里调用`Plugin.wrap()`方法，返回一个代理对象：
```java
public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
}
```
其中`wrap`方法又调用`Plugin.getSignatureMap`方法，获取`interceptor`中的注解，从而确定需要拦截的方法，然后放在Map中返回`wrap`方法。对于不匹配的对象，`wrap`方法直接返回该对象，否则创建一个代理实例。
```java
private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());
    }
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<Class<?>, Set<Method>>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.get(sig.type());
      if (methods == null) {
        methods = new HashSet<Method>();
        signatureMap.put(sig.type(), methods);
      }
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
}
```

也就是说，在`Interceptor`的执行顺序中，先通过`setProperties`方法获取配置文件中的参数（如果有）。然后所有类都经过`wrap`方法，对于符合拦截条件的，将经过`intercept`方法执行代理业务逻辑，否则直接返回。

## 处理SQL语句
已经确定了拦截的对象，下面我们要来考虑，可能调用`prepare()`方法的场景可能有很多，那么又怎样确认这个对象需要拦截呢？

这就需要我们在xml文件中配置的ID了。例如我们配置的sql语句id以`ByPage`结尾，这个语句执行的时候会触发`prepare`方法，也就是`prepare`方法是可以获取该SQL语句的id的。而当我们拦截`prepare`方法时，应该也是可以获取这个id的。当这个id符合约定的格式（例如这里是以`ByPage`结尾），即可确认该方法需要拦截。

我们从`BaseStatementHandler`类开始看起，其中`MappedStatement`对象所属的类中我们可以看到一些熟悉的属性，例如`private List<ResultMap> resultMaps;`、`private String id;`。虽然类中提供了get方法，但是在`BaseStatementHandler`中`MappedStatement`对象是protected声明的，我们既不与它同package，也没有继承关系，怎样获取这个id属性呢？答案只有反射。

MyBatis为我们提供了封装好的反射类`MetaObject`。
```java
public Object intercept(Invocation invocation) throws Throwable {
    StatementHandler statementHandler = (StatementHandler)invocation.getTarget();
    MetaObject metaObject = MetaObject.forObject(statementHandler, SystemMetaObject.DEFAULT_OBJECT_FACTORY, SystemMetaObject.DEFAULT_OBJECT_WRAPPER_FACTORY, new DefaultReflectorFactory());
}
```
这样我们就可以通过封装好的`metaObject`对象所提供的`getValue`方法来访问属性了。但是这里我们需要理清其中的继承关系。我们首先拦截到的是`RoutingStatementHandler`，然后是其中的`delegate`对象中的`mappedStatement`属性。
```java
public class RoutingStatementHandler implements StatementHandler {
  private final StatementHandler delegate;
}
```
因此我们的拦截器中应当这样获取SQL语句的id：
```java
MappedStatement mappedStatement = (MappedStatement)metaObject.getValue("delegate.mappedStatement");
// 配置文件中SQL语句的ID
String id = mappedStatement.getId();
if(id.matches(".+ByPage$")) {
    BoundSql boundSql = statementHandler.getBoundSql();
// 原始SQL语句
    String sql = boundSql.getSql();
}
```
这样我们就取到了原始的sql语句。那么现在我们要对SQL语句处理增加分页功能，除了原始的sql语句，还需要有什么？

没错，我们还需要SQL语句处理中产生的参数。因为statement中已经将参数替换为`?`等待赋值，此时需要了解参数的个数、类型、顺序等才能完成SQL语句的处理。
