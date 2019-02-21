---
title: MyBatis源码分析2-配置文件的处理
date: 2016-05-29 09:31:16
tags: [mybatis,xml]
categories:
- framework
- MyBatis
---

> MyBatis系列文章：
> https://zhum.in/blog/categories/frame/MyBatis/

> 项目代码：
> https://github.com/njZhuMin/AutoReplyRobot

- - -

# MyBatis拦截器
接着之前的往下。首先我们思考一下，要实现MySQL的分页查询功能，其实就在原始的sql语句后面拼接`LIMIT`语句。但是我们还需要配置参数。

## 获取参数
要完成`LIMIT`语句，我们需要总条数和page对象。
> 需要注意的是xml配置中`parameterType`只能传入一个类型参数，当有多个属性时，要么使用类进行封装，要么使用集合。这里我们用Map集合来传递两个参数。

```java
// 获得配置参数
Map<String, Object> parameter = (Map<String, Object>)boundSql.getParameterObject();
Page page = (Page)parameter.get("page");
// 带分页查询的SQL语句
String pageSql = sql + " LIMIT " + page.getDbIndex() + "," + page.getDbNumber();
// 修改原sql语句
metaObject.setValue("delegate.boundSql.sql", pageSql);
```

<!-- more -->

至此，我们已经完成了拦截和替换过程。使用`return invocation.proceed();`语句使被拦截的方法继续执行。实质上MyBatis中的`Invocation.proceed()`方法就是通过被拦截方法的反射使该方法继续执行：
```java
public class Invocation {
  	private Object target;
  	private Method method;
  	private Object[] args;

	public Object proceed() throws InvocationTargetException, IllegalAccessException {
    	return method.invoke(target, args);
	}
}
```

## 获取总条数
拦截器功能已经完成了，但是注意到page对象是不完整的。我们还需要获取查询的总条数来计算显示在前段页面的页数等属性。

查询语句很简单，只需要在原始SQL语句外嵌套`COUNT`的子查询：
```java
// 查询总条数SQL语句
String countSql = "SELECT COUNT(*) FROM(" + sql + ")a";
```
比较麻烦的是这个语句需要我们自己提交执行，类似于JDBC的流程。但是我们不用自己创建Connection，因为从拦截的对象中就可以获取Connection对象。
```java
Connection connection = (Connection)invocation.getArgs()[0];
```
但是别忘了这条SQL语句需要参数。因为此时检索条件中的参数已经被MyBatis替换成了`?`等待处理。但是既然时MyBatis解析的，那么它一定记录下了参数的顺序和值等信息。继续去`BaseStatementHandler`寻找：
```java
public abstract class BaseStatementHandler implements StatementHandler {
  	protected final ResultSetHandler resultSetHandler;
  	protected final ParameterHandler parameterHandler;
	//...
}
```
在`ParameterHandler`接口中又有如下方法的定义：
```java
public interface ParameterHandler {
	Object getParameterObject();
	void setParameters(PreparedStatement ps) throws SQLException;
}
```
由此可以看出，`setParameters`方法通过传入一个`PreparedStatement`来完成对它的赋值。在我们新组装的新SQL语句中，我们并没有改变参数的个数、顺序和类型，因此可以直接调用`setParameters`方法来为我们新的SQL语句完成赋值。
```java
PreparedStatement countStatement = connection.prepareStatement(countSql);
// 通过parameterHandler给PreparedStatement赋值
ParameterHandler parameterHandler = (ParameterHandler)metaObject.getValue("delegate.parameterHandler");
parameterHandler.setParameters(countStatement);
ResultSet rs = countStatement.executeQuery();

// 获得配置参数
Map<String, Object> parameter = (Map<String, Object>)boundSql.getParameterObject();
Page page = (Page)parameter.get("page");
if(rs.next()) {
page.setTotalNumber(rs.getInt(1));
```

最后，别忘了在MyBatis配置文件中注册插件：
```xml
<!-- 拦截器注册 -->
<plugins>
    <plugin interceptor="com.sunnywr.interceptor.PageInterceptor"></plugin>
</plugins>
```

# MyBatis配置文件的处理
## 问题提出
虽然数组和List集合可以相互转换，但是如果在MyBatis配置文件中的`parameterType`属性传入数组，会发生什么呢？

因为对于Java type，可以通过类名来获取它的Class，但是MyBatis又是怎样识别数组的呢？

## 总配置文件
我们首先从自定义的`DBAccess`类入手：
```java
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
```
其中的`build`方法调用了`SqlSessionFactoryBuilder`中的方法：
```java
public SqlSessionFactory build(Reader reader) {
    return build(reader, null, null);
}
```
而这个方法继续跟踪到：
```java
public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
try {
   	XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
   	return build(parser.parse());
	}
	// catch...
}
```
接着关注`parse()`方法，跟踪到`XMLConfigBuilder`类中：
```java
public class XMLConfigBuilder extends BaseBuilder {
  	private boolean parsed;

	public Configuration parse() {
		if (parsed) {
			throw new BuilderException("Each XMLConfigBuilder can only be used once.");
		}
		parsed = true;
		parseConfiguration(parser.evalNode("/configuration"));
		return configuration;
	}
}
```
变量`parsed`即保证了配置文件只被解析一次。因为这里解析的是MyBatis总配置文件，因此猜测`parser.evalNode("/configuration")`方法可以读取到总配置文件中引入的其他配置文件的路径并进而解析其中的标签。继续跟踪到`parser`对象所属的`XPathParser`类：
```java
public class XPathParser {
  	private Document document;

    public XPathParser(Reader reader) {
		commonConstructor(false, null, null);
		this.document = createDocument(new InputSource(reader));
 	}
}
```
MyBatis使用`DOM`方式解析xml文件，将Reader流转换成Document对象。其中使用`XPath`处理路径表达式：
```java
private Object evaluate(String expression, Object root, QName returnType) {
	try {
      	return xpath.evaluate(expression, root, returnType);
    } catch (Exception e) {
		throw new BuilderException("Error evaluating XPath.  Cause: " + e, e);
   }
}
```
这里的`Node`即是JDK中的标准类，并且可以继续执行查找Node中的子节点。
通过`XMLConfigBuilder`中的`parseConfiguration(parser.evalNode("/configuration"));`即读取了xml文件中`<configuration>`标签之间的配置信息保存在document对象中。

继续在`XMLConfigBuilder`类中可以看到很多类似的读取标签的方法：
```java
private void parseConfiguration(XNode root) {
    try {
      	Properties settings = settingsAsPropertiess(root.evalNode("settings"));
      	//issue #117 read properties first
      	propertiesElement(root.evalNode("properties"));
      	loadCustomVfs(settings);
      	typeAliasesElement(root.evalNode("typeAliases"));
      	pluginElement(root.evalNode("plugins"));
      	objectFactoryElement(root.evalNode("objectFactory"));
      	objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
		reflectorFactoryElement(root.evalNode("reflectorFactory"));
      	settingsElement(settings);
      	// read it after objectFactory and objectWrapperFactory issue #631
      	environmentsElement(root.evalNode("environments"));
      	databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      	typeHandlerElement(root.evalNode("typeHandlers"));
      	mapperElement(root.evalNode("mappers"));
	} catch (Exception e) {
      	throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
	}
}
```

## sql属性配置
其中`<mappers>`标签内配置了SQL语句的参数，因此我们来关注`mapperElement(root.evalNode("mappers"));`方法。跟踪`mapperElement`方法如下：
```java
private void mapperElement(XNode parent) throws Exception {
	if (parent != null) {
		for (XNode child : parent.getChildren()) {
        	if ("package".equals(child.getName())) {
				String mapperPackage = child.getStringAttribute("name");
				configuration.addMappers(mapperPackage);
        	} else {
				String resource = child.getStringAttribute("resource");
				String url = child.getStringAttribute("url");
				String mapperClass = child.getStringAttribute("class");
				if (resource != null && url == null && mapperClass == null) {
					ErrorContext.instance().resource(resource);
					InputStream inputStream = Resources.getResourceAsStream(resource);
					XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
					mapperParser.parse();
				} else if (resource == null && url != null && mapperClass == null) {
					ErrorContext.instance().resource(url);
					InputStream inputStream = Resources.getUrlAsStream(url);
					XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
					mapperParser.parse();
				} else if (resource == null && url == null && mapperClass != null) {
					Class<?> mapperInterface = Resources.classForName(mapperClass);
					configuration.addMapper(mapperInterface);
				} else {
					throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
				}
			}
		}
	}
}
```
关注`mapperParser.parse();`的解析方法：
```java
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      	configurationElement(parser.evalNode("/mapper"));
      	configuration.addLoadedResource(resource);
      	bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingChacheRefs();
    parsePendingStatements();
}
```
继续跟踪`configurationElement`方法：
```java
private void configurationElement(XNode context) {
    try {
    	String namespace = context.getStringAttribute("namespace");
		if (namespace == null || namespace.equals("")) {
			throw new BuilderException("Mapper's namespace cannot be empty");
		}
		builderAssistant.setCurrentNamespace(namespace);
		cacheRefElement(context.evalNode("cache-ref"));
		cacheElement(context.evalNode("cache"));
		parameterMapElement(context.evalNodes("/mapper/parameterMap"));
		resultMapElements(context.evalNodes("/mapper/resultMap"));
		sqlElement(context.evalNodes("/mapper/sql"));
		buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
	} catch (Exception e) {
		throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
	}
}
```
这里也印证了配置文件中必须配置`namespace`的原因。为了研究MyBatis解析数组的过程，我们继续跟踪`buildStatementFromContext`方法：
```java
private void buildStatementFromContext(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
		buildStatementFromContext(list, configuration.getDatabaseId());
	}
	buildStatementFromContext(list, null);
}
```
因为我们并没有配置`DatabaseId`，因此继续进入`buildStatementFromContext`方法的分支：
```java
private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
	for (XNode context : list) {
		final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
		try {
			statementParser.parseStatementNode();
		} catch (IncompleteElementException e) {
			configuration.addIncompleteStatement(statementParser);
		}
	}
}
```
这个方法中循环取到sql节点，关注`parseStatementNode`方法，其中找到对`parameterType`的处理方法：
```java
String parameterType = context.getStringAttribute("parameterType");
Class<?> parameterTypeClass = resolveClass(parameterType);
```
进入`resolveClass`方法：
```java
protected Class<?> resolveClass(String alias) {
    if (alias == null) {
    	return null;
    }
    try {
    	return resolveAlias(alias);
    } catch (Exception e) {
		throw new BuilderException("Error resolving class. Cause: " + e, e);
	}
}
```
因为我们传入的参数非空，因此继续看`resolveAlias`方法：
```java
protected Class<?> resolveAlias(String alias) {
    return typeAliasRegistry.resolveAlias(alias);
}
```

再跟踪`resolveAlias`方法：
```java
public <T> Class<T> resolveAlias(String string) {
    try {
    	if (string == null) {
        	return null;
		}
    	// issue #748
		String key = string.toLowerCase(Locale.ENGLISH);
		Class<T> value;
		if (TYPE_ALIASES.containsKey(key)) {
			value = (Class<T>) TYPE_ALIASES.get(key);
		} else {
			value = (Class<T>) Resources.classForName(string);
		}
		return value;
	} catch (ClassNotFoundException e) {
		throw new TypeException("Could not resolve type alias '" + string + "'.  Cause: " + e, e);
	}
}
```
首先将参数转换成小写，然后判断是否包含关键字。进入TYPE_ALIASES的构造函数：
```java
public class TypeAliasRegistry {

	private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<String, Class<?>>();

	public TypeAliasRegistry() {
		registerAlias("string", String.class);
		// ...
		registerAlias("int[]", Integer[].class);
		registerAlias("integer[]", Integer[].class);
		registerAlias("double[]", Double[].class);
		registerAlias("boolean[]", Boolean[].class);
	}

	public void registerAlias(String alias, Class<?> value) {
		if (alias == null) {
			throw new TypeException("The parameter alias cannot be null");
		}
		// issue #748
		String key = alias.toLowerCase(Locale.ENGLISH);
		if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
			throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
		}
		TYPE_ALIASES.put(key, value);
	}
}
```
观察到又一次转换成小写，然后存入Map中。这样看来，对于`parameterType`的解析依赖于构造函数中的定义。

同时`resolveAlias`方法中的最后，当在已定义的类型中找不到类型的时候，则通过`value = (Class<T>) Resources.classForName(string);`动态加载它的Class对象并返回。

从这里的源代码也可以看到，`parameterType`属性的取值是`大小写不敏感`的，对于`Map`类型也可以不用写全`java.util.Map`包名。

# 总结
## 批量新增的性能问题
当实现批量新增类似的功能时，使用JDBC最原始的方式通过循环：
```java
String insertSql = "insert into COMMAND_CONTENT(CONTENT,COMMAND_ID) values(?,?)";
PreparedStatement statement = conn.prepareStatement(insertSql);
for(CommandContent content : contentList) {
	statement.setString(1, content.getContent());
	statement.setString(2, content.getCommandId());
	statement.executeUpdate();
}
```
比较高效的方式是先批量提交最后一次执行：
```java
for(CommandContent content : contentList) {
	statement.setString(1, content.getContent());
	statement.setString(2, content.getCommandId());
	statement.addBatch();
}
statement.executeBatch();
```

同样的，使用MyBatis，初级的方式也是多次调用插入语句：
```java
DBAccess dbAccess = new DBAccess();
SqlSession sqlSession = null;
try {
	sqlSession = dbAccess.getSqlSession();
	// 通过sqlSession执行SQL语句
	ICommandContent commandContent = sqlSession.getMapper(ICommandContent.class);
	commandContent.insertBatch(contentList);
	sqlSession.commit();
}
```
而真正的批量新增，使用SQL语句的拼接来完成。对于MySql，SQL语句是这样的：
```sql
INSERT INTO table(column1,column2) VALUES(value1,1),(value2,2);
```
因此我们需要在MyBatis的xml中对VALUES后的括号进行拼接，注意使用
```xml
<insert id="insertBatch" parameterType="java.util.List">
	INSERT INTO command_content(CONTENT,COMMAND_ID) VALUES
    <foreach collection="list" item="item" separator=",">
		(#{item.content},#{item.commandId})
	</foreach>
</insert>
```
然后只需要执行一次查询即可：
```java
DBAccess dbAccess = new DBAccess();
SqlSession sqlSession = null;
try {
	sqlSession = dbAccess.getSqlSession();
	// 通过sqlSession执行SQL语句
	ICommandContent commandContent = sqlSession.getMapper(ICommandContent.class);
	commandContent.insertBatch(contentList);
	sqlSession.commit();
}
```

## DAO层的性能问题
在DAO层中有这样的语句：
```java
DBAccess dbAccess = new DBAccess();
```
而`DBAccess`类中完成的是配置文件的加载：
```java
public class DBAccess {
    public SqlSession getSqlSession() throws IOException {
        // 读取配置文件
        Reader reader = Resources.getResourceAsReader("Configuration.xml");
        // 构建SqlSessionFactory
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
        // 打开数据库会话
        SqlSession sqlSession = sqlSessionFactory.openSession();
        return sqlSession;
    }
}
```
> 这里有两个问题：

> 1. 配置文件要等到调用SQL语句时才被加载，这个时机有问题

> 2. 每次执行SQL的加载其实是相同的，配置文件重复加载影响性能

对于加载时机的问题，一个思路是定义一个`监听器`，在容器启动时完成加载。但是这样并不能解决重复加载的问题。那么对于这个问题，考虑使用`单例模式`的对象来实现配置的加载。

MyBatis的源码中也存在大量的判断重复加载的代码逻辑。

而在实际项目中，可以直接引入`Spring`框架。

## MyBatis在本案例中的特点
- SQL语句与代码分离
优点：便于管理维护
缺点：不便于调试，需要借助日志工具获得信息

- 用标签控制动态SQL语句的拼接
优点：用标签代替编写逻辑代码
缺点： 拼接复杂SQL语句时，没有代码灵活，比较复杂

- 结果集与Java对象的自动映射
优点：保证名称相同即可自动映射
缺点：对开发人员所写的SQL依赖性很强

- 编写原生SQL
优点：接近JDBC，很灵活
劣势：对SQL语句依赖程序很高，数据库移植不方便