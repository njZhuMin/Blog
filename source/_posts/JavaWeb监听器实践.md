---
title: Java Web监听器实践
date: 2016-05-14 13:46:13
tags: [Java,listener]
categories:
- Java
- listener
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

> 测试代码：
> https://github.com/njZhuMin/JavaDemo/tree/master/listener/CountListener

- - -

# 需求
使用监听器实现在线用户及人数的统计

> 1. 统计在线人数：使用`ServletSessionListener`监听器的初始化和销毁实现增加和删除

> 2. 在线用户信息：使用`ServletRequestListener`监听器的初始化实现获取用户信息

> 3. 保存：保存于全局的`getSession.getServletContext().getAttribute(attributeName, attributeValue`)里面

<!-- more -->

# 代码实现
## 用户信息JavaBean
```java
package com.sunnywr.entity;

public class User {
    private String sessionIdString;
    private String ipString;
    private String firstTimeString;

	//getters and setters ommitted
}
```

## 数量统计
我们通过实现一个`HttpSessionListener`，在`sessionCreated`和`sessionDestroyed`方法中引入计数器，实现数量统计。
```java
package com.sunnywr.listener;

import com.sunnywr.entity.User;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpSessionEvent;
import javax.servlet.http.HttpSessionListener;
import java.util.ArrayList;
import com.sunnywr.util.SessionUtil;

@WebListener
public class MyHttpSessionListener implements HttpSessionListener {
    private int userNumber = 0;

    public void sessionCreated(HttpSessionEvent se) {
        userNumber++;
        se.getSession().getServletContext().setAttribute("userNumber", userNumber);
    }

    public void sessionDestroyed(HttpSessionEvent se) {
        userNumber--;
        se.getSession().getServletContext().setAttribute("userNumber", userNumber);

        ArrayList<User> userList = null;
        userList = (ArrayList<User>)se.getSession().
                getServletContext().getAttribute("userList");
        if(SessionUtil.getUserBySessionId(userList, se.getSession().getId()) != null)
            userList.remove(SessionUtil.getUserBySessionId(userList, se.getSession().getId()));
    }
}
```

## 用户信息获取
通过实现`ServletRequestListener`来获取`request`对象，从而获取用户信息。将用户信息保存在`userList`中，并通过`session`的`setAttribute`和`getAttribute`方法传递。
```java
package com.sunnywr.listener;

import com.sunnywr.entity.User;
import javax.servlet.ServletRequestEvent;
import javax.servlet.ServletRequestListener;
import javax.servlet.annotation.WebListener;
import javax.servlet.http.HttpServletRequest;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Date;

import static com.sunnywr.util.SessionUtil.getUserBySessionId;
@WebListener
public class MyServletRequestListener implements ServletRequestListener {
    private ArrayList<User> userList;   //在线用户List

    public void requestDestroyed(ServletRequestEvent sre) {

    }

    public void requestInitialized(ServletRequestEvent sre) {
        userList = (ArrayList<User>) sre.getServletContext().getAttribute("userList");
        if(userList == null)
            userList = new ArrayList<User>();

        HttpServletRequest request = (HttpServletRequest)sre.getServletRequest();
        String sessionIdString = request.getSession().getId();
        if(getUserBySessionId(userList, sessionIdString) == null) {
            User user = new User();
            user.setSessionIdString(sessionIdString);
            user.setFirstTimeString(new
                    SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()));
            user.setIpString(request.getRemoteAddr());
            userList.add(user);
        }
        sre.getServletContext().setAttribute("userList", userList);
    }
}
```

## 用户信息与`Session`对象的关联
```java
package com.sunnywr.util;

import com.sunnywr.entity.User;
import java.util.ArrayList;

public class SessionUtil {
    public static Object getUserBySessionId(ArrayList<User> userList, String sessionIdString) {
        for(int i = 0; i < userList.size(); i++) {
            User user = userList.get(i);
            if(sessionIdString.equals(user.getSessionIdString()))
                return user;
        }
        return null;
    }
}
```

## JSP
```js
<%@ page import="java.util.ArrayList" %>
<%@ page import="com.sunnywr.entity.User" %>
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
<head>
    <title>UserInfo</title>
</head>
<body>
    Current online user number: ${userNumber} <br/>
    <%
        ArrayList<User> userList = (ArrayList<User>)
                request.getServletContext().getAttribute("userList");
        if(userList != null) {
            for(int i = 0; i < userList.size(); i++) {
                User user = userList.get(i);

    %>
    IP: <%=user.getIpString()%>, FirstTime: <%=user.getFirstTimeString()%>, SessionId: <%=user.getSessionIdString()%> <br/>
    <%}}%>
</body>
</html>
```
