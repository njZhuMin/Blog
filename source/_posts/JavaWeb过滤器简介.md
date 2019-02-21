---
title: Java Web过滤器简介
date: 2016-05-12 10:46:13
tags: [Java,filter]
categories:
- Java
- filter
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

> 测试代码：
> https://github.com/njZhuMin/JavaDemo/tree/master/filter/TestFilter

- - -

# 过滤器
## 定义
1. 过滤器三部分：过滤源（用户请求） -> 过滤规则 -> 过滤结果；

2. 过滤器不处理结果，只做辅助性操作；

3. 过滤器是一个服务器端的组件，它可以截取用户端的请求和响应信息，并对这些信息过滤。

## 生命周期
1. 在Web容器启动时依据web.xml实例化：一次

2. 初始化 init()：一次

3. 过滤 doFilter()：多次执行

4. 销毁 destroy()：一次，Web容器关闭

<!-- more -->

## 实现`javax.servlet.Filter`接口
- init()方法：这是过滤器的初始化方法，Web容器创建过滤器实例后将调用这个方法。这个方法中可以读取web.xml文件中过滤器的参数。

- doFilter()：这个方法完成实际的过滤操作。这个地方是过滤器的核心方法。当用户请求访问与过滤器关联的URL时，web容器将先调用过滤器的doFilter方法。FilterChain参数可以调用chain.doFilter方法，将请求传给下一个过滤器（或目标资源），或利用转发，重定向将请求转发到其他资源。

- destroy()：Web容器在销毁过滤器实例前调用该方法，在这个方法中可以释放过滤器占用的资源。（大多数情况用不到）

注意：
> 过滤器能改变用户请求的Web资源，能改变用户的请求的路径（如用户没登入可转到登入界面）

> 过滤器不能直接返回数据，不能直接处理用户请求。因为过滤器并不是一个标准的Servlet

# 过滤器链
当`web.xml`中存在多个过滤器，且它们的`url-pattern`相同的时候，就会按照`web.xml`中定义的先后顺序形成过滤器链。

例如我们使用`FirstFilter`与`SecondFilter`组装过滤器链：
```java
package com.sunnywr.filter;

import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

public class FirstFilter implements Filter {

    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("init FirstFilter");
    }

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("Start doFilter FirstFilter...");
        //chain.doFilter(request, response);

        HttpServletRequest req = (HttpServletRequest)request;
        HttpServletResponse res = (HttpServletResponse)response;
        //redirect
        res.sendRedirect(req.getContextPath() + "/main.jsp");
        //forward
        //req.getRequestDispatcher("main.jsp").forward(req, res);
        System.out.println("End doFilter FirstFilter...");
    }

    public void destroy() {
        System.out.println("destroy FirstFilter");
    }
}
```

```java
package com.sunnywr.filter;

import javax.servlet.*;
import java.io.IOException;

public class SecondFilter implements Filter{
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("init SecondFilter");
    }

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("Start doFilter SecondFilter...");
        chain.doFilter(request, response);
        System.out.println("End doFilter SecondFilter...");
    }

    public void destroy() {
        System.out.println("destroy SecondFilter");
    }
}
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://java.sun.com/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
         version="3.0">

    <welcome-file-list>
        <welcome-file>index.jsp</welcome-file>
    </welcome-file-list>

    <filter>
        <filter-name>FirstFilter</filter-name>
        <filter-class>com.sunnywr.filter.FirstFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>FirstFilter</filter-name>
        <url-pattern>/index.jsp</url-pattern>
        <dispatcher>FORWARD</dispatcher>
    </filter-mapping>
    <!--
    <filter>
        <filter-name>SecondFilter</filter-name>
        <filter-class>com.sunnywr.filter.SecondFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>SecondFilter</filter-name>
        <url-pattern>/main.jsp</url-pattern>
    </filter-mapping>
    -->
    <error-page>
        <error-code>404</error-code>
        <location>/404.jsp</location>
    </error-page>

    <filter>
        <filter-name>ErrorFilter</filter-name>
        <filter-class>com.sunnywr.filter.ErrorFilter</filter-class>
    </filter>
    <filter-mapping>
        <filter-name>ErrorFilter</filter-name>
        <url-pattern>/404.jsp</url-pattern>
        <dispatcher>ERROR</dispatcher>
    </filter-mapping>
</web-app>
```
使用Tomcat运行程序，可以看到对`index.jsp`的访问被重定向到`main.jsp`

# 过滤器分类
Web 2.5标准中定义了四种过滤器：`request`，`forward`，`include`和`error`。如果没有设置标签，那么默认标签是`request`。

## forward 与request 的区别
forward 指的是使用了服务器跳转时需要经过过滤器，而request表示使用客户端跳转时需要经过过滤器。

服务器跳转采用`<jsp:forward>`标签和`request.getRequestDispatcher("path").forward(request,response)`方式进行跳转，客户端跳转表示使用`response.sendRedirect()`方式进行跳转。

> 在jsp页面中使用forword标签和在servlet中使用的一样都是请求转发，如果过滤器设置了对请求转发行为的过滤，那么jsp页面中的请求转发一样会被过滤。

## dispatch参数
dispatch参数设定过滤器什么时候被激活。
```xml
<filter-mapping>
	<filter-name>FirstFilter</filter-name>
	<url-pattern>/main.jsp</url-pattern>
	<dispatcher>FORWARD</dispatcher>
</filter-mapping>
```

## ErrorFilter
```java
package com.sunnywr.filter;

import javax.servlet.*;
import java.io.IOException;

public class ErrorFilter implements Filter{
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("Error path");
        chain.doFilter(request, response);
    }

    public void destroy() {

    }
}
```

# Web 3.0中的异步支持
Web 3.0中引入了`ASYNC`类型的`Filter`以支持异步Servlet。异步Filter的使用需要在Servlet中开启`asyncSupported = true`，并且Filter中的`dispatcherTypes = {DispatcherType.REQUEST, DispatcherType.ASYNC}`。

使用注解的方式实现异步`AsyncFilter`：
```java
package com.sunnywr.servlet;

import javax.servlet.AsyncContext;
import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.Date;

@WebServlet(name = "AsyncServlet", value = "/servlet/AsyncServlet", asyncSupported = true)
public class AsyncServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {

    }

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        System.out.println("Servlet start at " + new Date());
        AsyncContext context = request.startAsync();
        new Thread(new Executor(context)).start();
        System.out.println("Servlet finish at " + new Date());
    }

    public class Executor implements Runnable {
        private AsyncContext context;

        public Executor(AsyncContext context){
            this.context = context;
        }
        public void run() {
            try {
                Thread.sleep(1000 * 10);

                //context.getRequest();
                //context.getResponse();
                System.out.println("Thread finish at " + new Date());
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

```java
package com.sunnywr.filter;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import java.io.IOException;

@WebFilter(filterName = "AsyncFilter", value = {"/index.jsp"},
        asyncSupported = true, dispatcherTypes = {DispatcherType.REQUEST, DispatcherType.ASYNC})
public class AsyncFilter implements Filter{
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        System.out.println("Start AsyncFilter...");
        chain.doFilter(request, response);
        System.out.println("End AsyncFilter...");
    }

    public void destroy() {

    }
}
```
