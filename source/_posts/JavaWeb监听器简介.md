---
title: Java Web监听器简介
date: 2016-05-13 18:46:13
tags: [Java,listener]
categories:
- Java
- listener
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

> 测试代码：
> https://github.com/njZhuMin/JavaDemo/tree/master/listener/ListenerDemo

- - -

# Web监听器
## 定义
监听器是`Servlet`规范中定义的一种特殊类：

- 用于监听`ServletContext`, `HttpSession`和`ServletRequest`等域对象的创建与销毁事件

- 用于监听域对象的属性发生修改的事件

- 在事件发生之前、发生后做一些必要的处理

## 用途
Web监听器的常见用途有：

- 统计在线人数和在线用户

- 系统启动时加载初始化信息

- 统计网站访问量

- 跟spring结合

<!-- more -->

## FirstListener
```java
import javax.servlet.ServletContextEvent;
import javax.servlet.ServletContextListener;
import javax.servlet.annotation.WebListener;

@WebListener
public class FirstListener implements ServletContextListener {
    public void contextInitialized(ServletContextEvent sce) {
        System.out.println("contextInitialized");
    }

    public void contextDestroyed(ServletContextEvent sce) {
        System.out.println("contextDestroyed");
    }
}
```
当Web应用启动的时候，会初始化并创建一个application实例，触发监听器打印`contextInitialized`，而应用结束的时候会调用application的destroy方法，触发打印`contextDestroyed`。

## 监听器启动顺序
监听器的启动顺序按照`web.xml`中的定义顺序依次启动。

如果`web.xml`中同时配置有`监听器`、`过滤器`和`Servlet`，那么它们按照优先级`监听器 > 过滤器 > Servlet`的顺序启动。

# 监听器的分类
## 按监听对象划分
- ServletContext：用于监听应用程序环境对象的事件监听器

- HttpSession：用于监听用户会话对象的事件监听器

- ServletRequest：用于监听请求消息对象的事件监听器

## 按监听事件划分
- 监听域对象自身的创建和销毁的事件监听器

- 监听域对象中的属性的增加和删除的事件监听器

- 监听绑定到HttpSession域中的某个对象的状态的事件监听器

## web.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">

    <listener>
        <listener-class>com.sunnywr.listener.FirstListener</listener-class>
    </listener>
    <listener>
        <listener-class>com.sunnywr.listener.MyHttpSessionAttributeListener</listener-class>
    </listener>
    <listener>
        <listener-class>com.sunnywr.listener.MyServletRequestAttributeListener</listener-class>
    </listener>
    <listener>
        <listener-class>com.sunnywr.listener.MyServletContextAttributeListener</listener-class>
    </listener>

    <context-param>
        <param-name>initParam</param-name>
        <param-value>testInitParam</param-value>
    </context-param>
    <session-config>
        <session-timeout>0</session-timeout>
    </session-config>
</web-app>
```

## ServletContextListener
ServletContextListener主要用于`定时器`和`全局属性对象`。
```java
public void contextInitialized(ServletContextEvent sce) {
	String initParam = sce.getServletContext().getInitParameter("initParam");
	System.out.println("contextInitialized : initParam = " + initParam);
}

public void contextDestroyed(ServletContextEvent sce) {
	System.out.println("contextDestroyed");
}
```

## HttpSessionListener
HttpSessionListener主要用于`统计在线人数`和`记录访问日志`。
```java
public void sessionCreated(HttpSessionEvent se) {
    System.out.println("sessionCreated" + new Date());
}

public void sessionDestroyed(HttpSessionEvent se) {
    System.out.println("sessionDestroyed" + new Date());
}
```
可以在`web.xml`中配置`session`的有效时间（单位：分钟）
```xml
<session-config>
    <session-timeout>1</session-timeout>
</session-config>
```

## ServletRequestListener
ServletRequestListener主要用于`读取参数`和`记录访问历史`。
```java
public void requestInitialized(ServletRequestEvent sre) {
    String name = sre.getServletRequest().getParameter("name");
    System.out.println("requestInitialized : name = " + name);
}

public void requestDestroyed(ServletRequestEvent sre) {
    System.out.println("requestDestroyed");
}
```

# 属性的增删改
## Listener
```java
package com.sunnywr.listener;

import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpSessionAttributeListener;
import javax.servlet.http.HttpSessionBindingEvent;

@WebListener
public class MyHttpSessionAttributeListener implements HttpSessionAttributeListener {
    public void attributeAdded(HttpSessionBindingEvent event) {
        System.out.println("HttpSession_attributeAdded : " + event.getName());
    }

    public void attributeRemoved(HttpSessionBindingEvent event) {
        System.out.println("HttpSession_attributeRemoved : " + event.getName());
    }

    public void attributeReplaced(HttpSessionBindingEvent event) {
        System.out.println("HttpSession_attributeReplaced : " + event.getName());
    }
}
```

```java
package com.sunnywr.listener;

import javax.servlet.ServletContextAttributeEvent;
import javax.servlet.ServletContextAttributeListener;
import javax.servlet.annotation.WebListener;

@WebListener
public class MyServletContextAttributeListener implements ServletContextAttributeListener {
    public void attributeAdded(ServletContextAttributeEvent event) {
        System.out.println("ServletContext_attributeAdded : " + event.getName());
    }

    public void attributeRemoved(ServletContextAttributeEvent event) {
        System.out.println("ServletContext_attributeRemoved : " + event.getName());
    }

    public void attributeReplaced(ServletContextAttributeEvent event) {
        System.out.println("ServletContext_attributeReplaced : " + event.getName());
    }
}
```

```java
package com.sunnywr.listener;

import javax.servlet.ServletRequestAttributeEvent;
import javax.servlet.ServletRequestAttributeListener;
import javax.servlet.annotation.WebListener;

@WebListener
public class MyServletRequestAttributeListener implements ServletRequestAttributeListener{
    public void attributeAdded(ServletRequestAttributeEvent srae) {
        System.out.println("ServletRequest_attributeAdded : " + srae.getName());
    }

    public void attributeRemoved(ServletRequestAttributeEvent srae) {
        System.out.println("ServletRequest_attributeRemoved : " + srae.getName());
    }

    public void attributeReplaced(ServletRequestAttributeEvent srae) {
        System.out.println("ServletRequest_attributeReplaced : " + srae.getName());
    }
}
```

## JSP
```js
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>init</title>
</head>
<body>
    <%
        request.setAttribute("requestName", "requestValue");
        request.getSession().setAttribute("sessionName", "sessionValue");
        request.getSession().getServletContext().setAttribute("contextName", "contextValue");
    %>
    Init attributes...<br/>
    <button onclick="location.href='<%=request.getContextPath()%>/init.jsp';">
        Init
    </button>
    <button onclick="location.href='<%=request.getContextPath()%>/destroy.jsp';">
        Destroy
    </button>
</body>
</html>
```

```js
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>destroy</title>
</head>
<body>
    <%
        request.removeAttribute("requestName");
        request.getSession().removeAttribute("sessionName");
        request.getSession().getServletContext().removeAttribute("contextName");
    %>
    Destroy attributes...<br/>
    <button onclick="location.href='<%=request.getContextPath()%>/init.jsp';">
        Init
    </button>
    <button onclick="location.href='<%=request.getContextPath()%>/destroy.jsp';">
        Destroy
    </button>
</body>
</html>
```

# 绑定到HttpSession域对象状态事件监听
## HttpSession的状态
- 绑定 HttpSession：setAttribute

- 解除绑定 HttpSesstion： removeAttribute

- 钝化：保存到持久化的对象中或者系统文件中

- 活化：从持久化的对象中反序列化取出

> 监听器类型：

> - HttpSessionBindingListener： HttpSession对象的绑定与解除绑定

>- HttpSessionActivationListener：Session对象的钝化与活化

## Session的钝化机制
Session的钝化机制为了解决高并发状态下的服务器性能问题。

钝化机制的本质是：服务器将不经常使用的Session对象暂时序列化到系统文件系统或数据库系统中，当使用时再反序列化到内存中。整个过程由服务器自动完成。

Tomcat中两种Session钝化管理器：

- 首先session钝化机制是由sessionManager管理

 - StandarManager：`org.apache.catalina.session.StandarManager`

   1.当Tomcat服务器关闭或者重启时，tomcat服务器会将当前内存中的session对象钝化到服务器文件系统中

   2.另一种情况是web应用程序被重新加载时，内存中的session对象也会被钝化到服务器的文件系统中

 - Persistentmanager：`   org.apache.catalina.session.Persistentmanager`

   除了上述两种情况外，`Persistentmanager`还对应处理第3种情况：

   可以配置主流内存的session对象数目，将不经常使用的session对象保存到系统

默认情况下，Tomcat提供2个钝化驱动类：`HttpServletBindingListener`、
`HttpSessionActionListener`。

当对象实现Persistentmanager的接口之后，这个对象被session绑定了，这时会触发事件，执行方法。

> `HttpServletBindingListener`和`HttpSessionActionListener`是默认对象，不需要做监听器声明。

## 代码实现
`HttpServletBindingListener`和`HttpSessionActionListener`是对JavaBean的接口，因此实现类似于JavaBean。
```java
package com.sunnywr.entity;

import javax.servlet.http.HttpSessionActivationListener;
import javax.servlet.http.HttpSessionBindingEvent;
import javax.servlet.http.HttpSessionBindingListener;
import javax.servlet.http.HttpSessionEvent;
import java.io.Serializable;

public class User implements HttpSessionBindingListener, HttpSessionActivationListener,
        //session钝化需要实现Serializable接口
        Serializable {
    private String username;
    private String password;

    public void valueBound(HttpSessionBindingEvent event) {
        System.out.println("valueBound name : " + event.getName());
    }

    public void valueUnbound(HttpSessionBindingEvent event) {
        System.out.println("valueUnbound name : " + event.getName());
    }

    public void sessionWillPassivate(HttpSessionEvent se) {
        System.out.println("sessionWillPassivate : " + se.getSource());
    }

    public void sessionDidActivate(HttpSessionEvent se) {
        System.out.println("sessionDidActivate : " + se.getSource());
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```
可以看到，当session实例化的时候，完成了属性值的`bound`操作。而当直接停止tomcat服务器时，当前的`session`对象被序列化钝化保存为`org.apache.catalina.session.StandarManager`对象。
