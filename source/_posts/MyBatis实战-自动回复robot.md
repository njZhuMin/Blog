---
title: MyBatis实战-自动回复robot
date: 2016-05-27 09:31:16
tags: [mybatis,log4j]
categories:
- framework
- MyBatis
---

> MyBatis系列文章：
> https://zhum.in/blog/categories/frame/MyBatis/

> 项目代码：
> https://github.com/njZhuMin/AutoReplyRobot

- - -

# JDBC Demo
话不多说，先上一个JDBC版本的Demo，然后我们在这个版本基础上使用MyBatis重构。

```bash
git clone https://github.com/njZhuMin/AutoReplyRobot/tree/jdbc
```

以下总结一些JDBC版本中的需要注意的细节和坑。

# 事务流程
```bash
前端JSP向服务器发送请求--> Servlet获取请求 --> Service解析请求 --> 调用DAO层查询 --> JDBC返回查询结果
```

<!-- more -->

## JSP
1. 页面跳转需要由 Servlet 控制逻辑，因此页面必须放在 WEB-INF 下防止被直接访问。

2. 如何获取资源路径
```js
String path = request.getContextPath();
String basepath = request.getScheme() + "://" + request.getServerName() + ":" + request.getServerPort() + path + "/";
```

3. 使用 JSTL 增加页面开发效率
```js
<c:forEach items="${messageList}" var="message" varStatus="status">
	/* 隔行设置背景色 */
	<tr <c:if test="${status.index % 2 != 0}">
		style="background-color:#ECF6EE;"
		</c:if> >
		<td><input type="checkbox" /></td>
		<td>${status.index + 1}</td>
        <td>${message.command}</td>
        <td>${message.description}</td>
        <td>
        	<a href="#">修改</a>&nbsp;&nbsp;&nbsp;
        	<a href="#">删除</a>
        </td>
	</tr>
</c:forEach>
```

## Servlet控制层
Servlet作为控制层，一般处理的事务：接受参数、向页面传参、调用Service层服务、控制页面跳转。

```java
package com.sunnywr.servlet;

import com.sunnywr.service.ListService;

import javax.servlet.ServletException;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

/**
 * 内容列表页面控制
 */
public class ListServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        // 设置编码
        req.setCharacterEncoding("UTF-8");
        // 接受页面参数
        String command = req.getParameter("command");
        String description = req.getParameter("description");
        // 向页面传参数
        req.setAttribute("command", command);
        req.setAttribute("description", description);

        ListService listService = new ListService();
        // 查询消息列表并传给页面
        req.setAttribute("messageList", listService.queryMessageList(command, description));
        // 页面跳转
        req.getRequestDispatcher("/WEB-INF/jsp/back/list.jsp").forward(req, resp);
    }

    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        this.doGet(req, resp);
    }
}
```

## Service层
```java
package com.sunnywr.service;

import com.sunnywr.bean.Message;
import com.sunnywr.dao.MessageDao;

import java.util.List;

/**
 * 列表相关的业务
 */
public class ListService {
    public List<Message> queryMessageList(String command, String description) {
        MessageDao messageDao = new MessageDao();
        return messageDao.queryMessageList(command, description);
    }
}
```

## DAO层
> SQL处理的一些细节和坑

1. dataSource需要加编码，否则查询结果会出现乱码。
```java
Class.forName("com.mysql.jdbc.Driver");
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/micro_message?characterEncoding=utf8", "root", "root");
```
调试过程中出现乱码，一开始以为是 JSP、Servlet 或者 Tomcat 中的编码问题，各种改编码，甚至对所有request的String重新encoding，就是不对。后来碰巧发现是数据库查询时的问题。

2. 多条件查询的处理

- 使用StringBuilder拼接多个查询条件时，避免使用`String +=`，这会产生多个String常量实例，必须依赖GC自动回收，造成安全隐患

- 多线程环境中使用`StringBuffer`保证线程安全

- 条件拼接技巧：`" WHERE 1=1 AND conditions 1 AND condition 2"`

- 查询时不要使用`SELECT *`语句，需要数据库引擎对列做解析，影响效率

- 使用PreparedStatement防止SQL注入，使用List缓存多个条件

- 集合中存放的是对象的引用，可以先add后修改

```java
StringBuilder sql = new StringBuilder("SELECT ID, COMMAND, DESCRIPTION, CONTENT FROM message WHERE 1=1");

// 缓存查询条件
List<String> paramList = new ArrayList<String>();
if(command != null && !"".equals(command.trim())) {
	sql.append(" AND COMMAND= ? ");
	paramList.add(command);
}
// 模糊查询
if(description != null && !"".equals(description.trim())) {
	sql.append(" AND DESCRIPTION LIKE '%' ? '%'");
	paramList.add(description);
}
// 遍历拼接条件
PreparedStatement statement = conn.prepareStatement(sql.toString());
for(int i = 0; i < paramList.size(); i++) {
    statement.setString(i+1, paramList.get(i));
}

ResultSet rs = statement.executeQuery();
while(rs.next()) {
	Message message = new Message();
    messageList.add(message);
    message.setId(rs.getString("ID"));
   	message.setCommand(rs.getString("COMMAND"));
    message.setDescription(rs.getString("DESCRIPTION"));
    message.setContent(rs.getString("CONTENT"));
}
```

# 基于MyBatis重构
MyBatis提供`SqlSession`对象，该对象允许：

1. 向 SQL 语句传入参数
2. 执行SQL语句
3. 获取执行SQL语句的结果
4. 事务的控制

## 核心配置文件
- Configuration.xml
```xml
<!DOCTYPE configuration
    PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
	<environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"></transactionManager>
            <dataSource type="UNPOOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/micro_message?characterEncoding=utf8"/>
                <property name="username" value="root"/>
                <property name="password" value="root" />
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="sqlXml/User.xml"/>
    </mappers>
</configuration>
```

- User.xml
```xml
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="Message">
    <resultMap type="com.sunnywr.bean.Message" id="MessageResult">
        <id column="ID" jdbcType="INTEGER" property="id"/>
        <result column="COMMAND" jdbcType="VARCHAR" property="command"/>
        <result column="DESCRIPTION" jdbcType="VARCHAR" property="description"/>
        <result column="CONTENT" jdbcType="VARCHAR" property="content"/>
    </resultMap>

    <select id="queryMessageList" parameterType="long" resultMap="MessageResult">
        SELECT ID, COMMAND, DESCRIPTION, CONTENT FROM message WHERE 1=1
    </select>
</mapper>
```

- 读取核心配置文件
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

## 动态拼接SQL
配置文件中使用OGNL表达式：
```xml
<!DOCTYPE mapper
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="Message">
    <resultMap type="com.sunnywr.bean.Message" id="MessageResult">
        <id column="ID" jdbcType="INTEGER" property="id"/>
        <result column="COMMAND" jdbcType="VARCHAR" property="command"/>
        <result column="DESCRIPTION" jdbcType="VARCHAR" property="description"/>
        <result column="CONTENT" jdbcType="VARCHAR" property="content"/>
    </resultMap>

    <select id="queryMessageList" parameterType="com.sunnywr.bean.Message" resultMap="MessageResult">
        SELECT ID, COMMAND, DESCRIPTION, CONTENT FROM message WHERE 1=1
        <if test="command != null and !&quot;&quot;.equals(command.trim())">
            AND COMMAND=#{command}
        </if>
        <if test="description != null and !&quot;&quot;.equals(description.trim())">
            AND DESCRIPTION LIKE '%' #{description} '%'
        </if>
    </select>
</mapper>
```

## 使用Log4j调试动态SQL
MyBatis本身提供了对 Log4j 日志系统的整合，可以向日志输出SQL语句组装以及查询过程。

首先在工程中引入 log4j 依赖，然后添加配置文件`log4j.properties`：

```xml
log4j.rootLogger=DEBUG,Console
log4j.appender.Console=org.apache.log4j.ConsoleAppender
log4j.appender.Console.layout=org.apache.log4j.PatternLayout
log4j.appender.Console.layout.ConversionPattern=%d [%t] %-5p %c - %m%n
log4j.logger.org.apache=INFO
```

重新启动tomcat，执行查询输出日志如下：
```bash
2016-08-08 20:05:47,929 [http-nio-8080-exec-3] DEBUG Message.queryMessageList - ==>  Preparing: SELECT ID, COMMAND, DESCRIPTION, CONTENT FROM message WHERE 1=1 AND DESCRIPTION LIKE '%' ? '%'
2016-08-08 20:05:47,977 [http-nio-8080-exec-3] DEBUG Message.queryMessageList - ==> Parameters: 精彩(String)
2016-08-08 20:05:48,012 [http-nio-8080-exec-3] DEBUG Message.queryMessageList - <==      Total: 2
```

# 删除与自动回复
## 删除记录
单条记录的删除：
```bash
Form提交id --> sqlSession拼接SQL语句 --> sqlSession.commit() --> 页面跳转List.action重新加载列表
```

多条记录的删除：
```bash
Form复选框提交id数组 --> sqlSession拼接SQL语句 --> sqlSession.commit() --> 页面跳转List.action重新加载列表
```

MyBatis对应映射
```xml
<delete id="deleteOne" parameterType="int">
    DELETE FROM message WHERE ID=#{_parameter}
</delete>
<delete id="deleteMulti" parameterType="java.util.List">
    DELETE FROM message WHERE ID IN(
    <foreach collection="list" item="item" separator=",">
        #{item}
    </foreach>
    )
</delete>
```

根据分层原则，Servlet只负责传参与控制跳转，所有对数据的处理应留给Service层。
```java
public class MaintainService {
    /**
     * 单条记录删除
     */
    public void deleteOne(String id) {
        if(id != null && !"".equals(id.trim())) {
            MessageDao messageDao = new MessageDao();
            messageDao.deleteOne(Integer.valueOf(id));
        }
    }

    /**
     * 多条记录删除
     */
    public void deleteMulti(String[] ids) {
        List<Integer> idList = new ArrayList<Integer>();
        for(String id : ids)
            idList.add(Integer.valueOf(id));
        MessageDao messageDao = new MessageDao();
        messageDao.deleteMulti(idList);
    }
}
```

# 一对多回复
## 表设计
将指令表拆分成两张，即满足数据库设计范式，也满足一对多的查询需求。
```sql
CREATE TABLE command (
    ID INT PRIMARY KEY AUTO_INCREMENT,
    NAME VARCHAR(16),
    DESCRIPTION VARCHAR(32)
) ENGINE=InnoDB CHARSET=utf8;

CREATE TABLE command_content (
    ID INT PRIMARY KEY,
    CONTENT VARCHAR(2048),
    COMMAND_ID INT
) ENGINE=InnoDB CHARSET=utf8;
```

## MyBatis Mapper
- 一对多的映射关系时，使用`collection`元素映射

- 多个表出现相同列名时需要使用别名，这是因为MyBatis和JDBC中相同，使用`getMetaData()`方法不能获取到表名。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="Command">
    <resultMap type="com.sunnywr.bean.Command" id="Command">
        <id column="C_ID" jdbcType="INTEGER" property="id"/>
        <result column="NAME" jdbcType="VARCHAR" property="name"/>
        <result column="DESCRIPTION" jdbcType="VARCHAR" property="description"/>
        <collection property="contentList"  resultMap="CommandContent.Content"/>
    </resultMap>

    <select id="queryCommandList" parameterType="com.sunnywr.bean.Command" resultMap="Command">
        SELECT a.ID C_ID,a.NAME,a.DESCRIPTION,b.ID,b.CONTENT,b.COMMAND_ID
        FROM COMMAND a LEFT JOIN COMMAND_CONTENT b
        ON a.ID=b.COMMAND_ID
        <where>
            <if test="name != null and !&quot;&quot;.equals(name.trim())">
                AND a.NAME=#{name}
            </if>
            <if test="description != null and !&quot;&quot;.equals(description.trim())">
                AND a.DESCRIPTION LIKE '%' #{description} '%'
            </if>
        </where>
    </select>
</mapper>
```

# 一些容易混淆的概念和总结
## resultType和resultMap
- `resultMap`对应`xml`中的`resultMap`标签id，需要先在`resultMap`标签配置表与Java实体类的映射关系

- `resultType`对应Java中的类型，通过名称将列名与Java实体类属性对应，并且是大小写不敏感的

## #{}和${}
- `#{}`在MyBatis处理中，会先被替换成`?`，然后参与SQL的预编译。SQL预编译使SQL动态拼接变得简单方便，也有利于防止SQL注入

- `${}`在MyBatis处理中会被直接替换，例如`ORDER BY ${columnName}`，会被替换成`OEDER BY 列名`，可以发现这样的语法是错误的。因此在使用`${}`时要注意语法的匹配问题：`ORDER BY '${columnName}'`

## 异常处理
- 如果没有在MyBatis总配置文件中引入实体类的xml配置文件，会抛出`does not contain value for someClass`异常

- 如果配置中出现冲突的id，会抛出`namespace.id`异常

- SQL语法错误，在`log4j`日志输出时会抛出`error in SQL syntax`异常
