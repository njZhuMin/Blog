---
title: Java Web过滤器实践
date: 2016-05-13 10:46:13
tags: [Java,filter]
categories:
- Java
- filter
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

> 测试代码：
> https://github.com/njZhuMin/JavaDemo/tree/master/filter/LoginFilter

- - -

# 过滤器应用场景
过滤器在实际项目中一般的应用场景有：

> 1. 对用户请求进行统一认证

> 2. 编码格式转换

> 3. 对用户发送的数据进行过滤替换

> 4. 转换图像格式

> 5. 对响应的内容进行压缩

<!-- more -->

# 登录Servlet案例
## JSP
- login.jsp
```js
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Login</title>
</head>
<body>
    <form action="<%=request.getContextPath() %>/servlet/LoginServlet" method="post">
        UserName: <input type="text" name="username" /><br/>
        Password: <input type="password" name="password" /><br/>
        <input type="submit" value="Submit" />
    </form>
</body>
</html>
```

- success.jsp
```js
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Success</title>
</head>
<body>
    Login success. Welcome, ${username}.
</body>
</html>
```

- fail.jsp
```js
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>Fail</title>
</head>
<body>
    Login fail. Check again.
</body>
</html>
```

## LoginServlet.java

```java
package com.sunnywr.servlet;

import javax.servlet.ServletException;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
import java.io.IOException;

@WebServlet(name = "LoginServlet", value = "/servlet/LoginServlet")
public class LoginServlet extends HttpServlet {
    protected void doPost(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {
        String username = request.getParameter("username");
        String password = request.getParameter("password");
        if("admin".equals(username) && "admin".equals(password)) {
            //校验通过
            HttpSession session = request.getSession();
            session.setAttribute("username", username);

            response.sendRedirect(request.getContextPath() + "/success.jsp");
        } else {
            //校验失败
            response.sendRedirect(request.getContextPath() + "/fail.jsp");
        }
    }

    protected void doGet(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {

    }
}
```

## 过滤器的使用
我们通过`LoginServlet`的`doPost`方法，对于通过登录校验的用户，把`username`属性放进Session，然后在JSP页面显示出来。

然而这样是不安全的。因为我们仍可以通过浏览器直接访问`/success.jsp`页面。因此我们需要设置一个过滤器来过滤用户请求。

```java
package com.sunnywr.filter;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
import java.io.IOException;

@WebFilter(filterName = "LoginFilter", value = {"/success.jsp"})
public class LoginFilter implements Filter {
    public void destroy() {
    }

    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException, IOException {
        HttpServletRequest request = (HttpServletRequest)req;
        HttpServletResponse response = (HttpServletResponse)resp;
        HttpSession session = request.getSession();

        if(session.getAttribute("username") != null) {
            chain.doFilter(req, resp);
        } else {
            response.sendRedirect("login.jsp");
        }
    }

    public void init(FilterConfig config) throws ServletException {

    }
}
```

# 问题与改进
## 改进1
我们可以看到，对`success.jsp`过滤后，通过浏览器无法直接访问`success.jsp`了。当我们输入非法的用户名和密码之后，也被重定向到`fail.jsp`的页面。

但是在实际应用中，我们绝大部分的网页可能都需要登录才能展示，这样就必须在过滤器中增加无数排除条件。这显然是不可行的。因此我们考虑使用`/*`来匹配拦截器，但对`login.jsp`放行，否则请求被过滤器拦截，再次跳转回`login.jsp`页面。这样就陷入了循环。
```java
if(request.getRequestURL().indexOf("login.jsp") != -1) {
	chain.doFilter(req, resp);
	return;
}
```
运行后发现当我们输入用户名密码后，`/servlet/login.jsp`页面跳转再次被过滤器拦截。因此我们再继续添加Servlet的排除条件：
```java
if(request.getRequestURL().indexOf("login.jsp") != -1
		|| request.getRequestURL().indexOf("servlet") != -1) {
	chain.doFilter(req, resp);
	return;
}
```
等我们输入正确的用户名和密码之后，由于当前`session`存在`username`属性，因此请求`success.jsp`被过滤器放行。

## 改进2
此时关闭当前窗口，打开一个新的窗口，访问`success.jsp`，由于新的`session`中`username`属性不存在，因此请求被重定向回`login.jsp`。但是当我们输入错误登录信息时，并没有跳转到`fail.jsp`页面。因为`fail.jsp`请求又被拦截了。那么我们又需要向`LoginFilter`中写入排除条件。

这样显然不利于开发与维护。那么我们考虑使用`FilterConfig`对象进行初始化：
```java
package com.sunnywr.filter;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.annotation.WebInitParam;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;
import java.io.IOException;

@WebFilter(filterName = "LoginFilter", value = {"/*"},
        initParams = @WebInitParam(name = "noFilterPath", value = "login.jsp;fail.jsp;servlet"))
public class LoginFilter implements Filter {
    private FilterConfig config;
    public void destroy() {
    }

    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException, IOException {
        HttpServletRequest request = (HttpServletRequest)req;
        HttpServletResponse response = (HttpServletResponse)resp;
        HttpSession session = request.getSession();

        String noFilterPath = config.getInitParameter("noFilterPath");
        if(noFilterPath != null) {
            String[] strArray = noFilterPath.split(";");
            for(String path : strArray) {
                if(path == null || "".equals(path))
                    continue;
                if(request.getRequestURL().indexOf(path) != -1) {
                    chain.doFilter(req, resp);
                    return;
                }
            }
        }

        if(session.getAttribute("username") != null) {
            chain.doFilter(req, resp);
        } else {
            response.sendRedirect("login.jsp");
        }
    }

    public void init(FilterConfig config) throws ServletException {
        this.config = config;
    }

}
```
将注解中的`@WebInitParam(name = "noFilterPath", value = "login.jsp;fail.jsp;servlet")`参数使用`web.xml`进行配置，即可做到扩展性配置而无需改动代码。

# 使用过滤器进行编码转换
在`LoginFilter`中，我们只需要执行
```java
request.setCharacterEncoding("UTF-8");
```
即可完成字符编码的转换。
但是为了便于配置，我们在`web.xml`中增加`initParam`配置项，即可实现可扩展的编码转换。
```java
@WebFilter(filterName = "LoginFilter", value = {"/*"},
    initParams = {
        @WebInitParam(name = "noFilterPath", value = "login.jsp;fail.jsp;servlet"),
        @WebInitParam(name = "charset", value = "UTF-8")})
public class LoginFilter implements Filter {
    private FilterConfig config;
	//function destroy and init ommitted...

	public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain) throws ServletException, IOException {
        //pre content ommitted...
		String charset = config.getInitParameter("charset");
        //未指定则使用默认编码
        if(charset == null)
            charset = "UTF-8";
        request.setCharacterEncoding(charset);
```
