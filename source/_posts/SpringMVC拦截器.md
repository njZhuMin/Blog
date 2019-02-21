---
title: SpringMVC拦截器
date: 2016-06-16 09:31:16
tags: [SpringMVC,controller,filter]
categories:
- framework
- SpringMVC
---

> SpringMVC系列文章：
> http://zhum.in/blog/categories/frame/SpringMVC/

> 示例代码：
> https://github.com/njZhuMin/SpringMVCDemo

- - -

# MVC与前端控制器
## 前端控制器
我们先来看一下SpringMVC中前端控制器的流程图：

{% asset_img front_controller.png front_controller %}

一个访问请求到达前端控制器，请求控制器得到业务数据，返还给前端控制器，前端控制器再把业务数据分发给业务视图，视图来呈现业务页面返还给前端控制器，前端控制器再呈现给浏览器。

> MVC的核心思想是业务数据抽取和业务数据呈现相分离

<!-- more -->

## MVC架构模式
MVC是一种使用`模型-视图-控制器`设计创建Web应用程序的模式：

- Model（模型）表示应用程序核心（比如数据库记录列表）

- View（视图）显示数据（数据库记录）

- Controller（控制器）处理输入（写入数据库记录）

# SpringMVC基本概念
## 静态概念
- DispatherServlet
SpringMVC中的`DispatherServlet`就是前端控制器的实现。请求通过Cotroller提交给DispatherServlet，再分发给Model和View层，实现业务的传递。

- Controller
`Cotroller`根据提交的请求完成对Model的生成。

- HandlerAdapter
SpringMVC中的`HandlerAdapter`在`DispatherServlet`中被使用，实际上就是Controller的表现形式。
SpringMVC中没有类似的Controller接口，通过适配器模式实现`HandlerAdapter`对`Controller`的调用。

- HandlerInterceptor
SpringMVC中提供的拦截器接口，可以在调用Controller前后进行拦截。

- HandlerMapping
`HandlerMapping`实现`DispatherServlet`与`Controller`之间的实现，向DispatherServlet返回封装好的HandlerAdapter对象。

- HandlerExecutionChain
封装`Controller`与`HandlerInterceptor`，构成执行链：
```bash
preHandle --> Controller method --> postHandle --> afterCompletion
```

- ModelAndView
SpringMVC中对`Model`的表现形式。

- ViewResolver
视图解析器，根据配置找到需要的视图对象

- View
完成页面呈现的类

## MVC动态概念
{% asset_img mvc.png mvc %}

在整个流程中，我们重点只需要关注`Controller`和`HandlerInterceptor`的实现。

## Web工程中的配置层级
{% asset_img config.png config %}

- `web.xml`是所有的web项目都要有的最最基本的配置文件，所有框架、监听器、Servlet等都在这里配置。
- `springmvc-dispatcher-servlet.xml`配置SpringMVC控制层的相关配置。
- `applicationContext.xml`这个是Spring Bean容器的配置文件，配置Spring中的IoC与AOP。

web.xml作为底层配置，监听SpringMVC的前端控制器，SpringMVC将请求分发给控制器后调用Spring容器进行业务处理，加载Spring配置文件。

## SpringMVC三种形式拦截处理
### 方式一：GET参数
```java
// 拦截 /courses/view?courseId=123 形式的URL
@RequestMapping(value = "/view", method = RequestMethod.GET)
public String viewCourse(@RequestParam("courseId") Integer courseId, Model model) {
    logger.debug("In viewCourse, courseId = {}", courseId);
    Course course = courseService.getCoursebyId(courseId);
    model.addAttribute(course);
    return "course_overview";
}
```
### 方式二：RESTFUL地址参数
```java
// 拦截 /courses/view2/123 形式的URL
@RequestMapping("/view2/{courseId}")
public String viewCourse2(@PathVariable("courseId") Integer courseId,
                          Map<String, Object> model) {
    logger.debug("In viewCourse2, courseId = {}", courseId);
    Course course = courseService.getCoursebyId(courseId);
    model.put("course", course);
    return "course_overview";
}
```
### 方式三：Servlet请求
```java
//拦截 /courses/view3?courseId=123 形式的URL
@RequestMapping("/view3")
public String viewCourse3(HttpServletRequest request) {
    Integer courseId = Integer.valueOf(request.getParameter("courseId"));
    Course course = courseService.getCoursebyId(courseId);
    logger.debug("In viewCourse3, courseId = {}", courseId);
    request.setAttribute("course",course);
    return "course_overview";
}
```

# SpringMVC文件上传
依赖`commons-fileupload`组件
```java
@RequestMapping(value="/upload", method=RequestMethod.GET)
public String showUploadPage() {
    return "course_admin/file";
}

@RequestMapping(value="/doUpload", method=RequestMethod.POST)
public String doUploadFile(@RequestParam("file") MultipartFile file) throws IOException {
    if(!file.isEmpty()) {
        logger.debug("Process file: {}", file.getOriginalFilename());
        FileUtils.copyInputStreamToFile(file.getInputStream(),
                new File("/home/silverlining/",
                System.currentTimeMillis() + file.getOriginalFilename()));
    }
    return "success";
}
```

# 拦截器
## 什么是拦截器
拦截器一般指的是在浏览器页面向服务端发出请求后，拦截请求，对请求进行一系列的操作；或者在服务器返回数据时，在数据到达浏览器界面前，做一些操作。

拦截器一般用于权限验证、乱码处理等操作。

## 拦截器的简单实现
1. 编写拦截器类实现`HandlerInterceptor`接口
```java
public class MyInterceptor implements HandlerInterceptor {
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("preHandle...");
        return true;
    }

    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("postHandle...");
    }

    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("afterCompletion...");
    }
}
```

2. 将拦截器注册到SpringMVC
```xml
<!-- 注册拦截器 -->
<mvc:interceptors>
    <bean id="myInterceptor" class="com.sunnywr.interceptor.MyInterceptor"></bean>
</mvc:interceptors>
```

3. 配置拦截器的拦截规则
```xml
<mvc:interceptors>
    <mvc:interceptor>
        <mvc:mapping path="/viewAll.form"/>
        <bean id="myInterceptor" class="com.sunnywr.interceptor.MyInterceptor"></bean>
    </mvc:interceptor>
</mvc:interceptors>
```

## HandlerInterceptor接口
- preHandle
是否将当前请求拦截下来，返回true请求继续运行，返回false请求终止（包括action层也会终止）。Object arg代表被拦截的请求的目标对象。

- postHandle
`ModelAndView arg`可以改变显示的视图，或修改发往视图的信息方法

- afterCompletion
表示视图显示之后在执行该方法，一般用于资源的销毁

## 多个拦截器的应用
当存在多个拦截器时，执行顺序如下：
{% asset_img interceptor.png interceptor %}

## 拦截器的其他实现方式
我们还可以通过实现`WebRequestInterceptor`接口来实现拦截器：
```java
public class MyInterceptor2 implements WebRequestInterceptor {
    public void preHandle(WebRequest request) throws Exception {

    }

    public void postHandle(WebRequest request, ModelMap model) throws Exception {

    }

    public void afterCompletion(WebRequest request, Exception ex) throws Exception {

    }
}
```
可以看到这个接口的实现与`HandlerInterceptor接口`类似，但是其中的`preHandle()`方法没有返回值，因此不能通过返回值决定是否终止请求。

## 拦截器的应用场景
拦截器不需要在web.xml中进行配置。使用原则是一般用来处理所有请求的共同问题。例如：

1. 解决乱码问题
```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    // 设置编码
    request.setCharacterEncoding("utf-8");
    return true;
}
```

2. 解决权限验证问题
```java
public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
    //权限验证
    if(request.getSession().getAttribute("user") == null) {
        request.getRequestDispatcher("/login.jsp").forward(request, response);
        return false;
    }
    return true;
}
```

## 拦截器与过滤器的区别
1. 拦截器是基于Java的反射机制的，而过滤器是基于函数回调。

2. 拦截器不依赖与Servlet容器，过滤器依赖与Servlet容器。

3. 拦截器只能对action请求起作用，而过滤器则可以对几乎所有的请求起作用（可过滤资源）。

4. 拦截器可以访问action上下文、值栈里的对象，而过滤器不能访问。

5. 在action的生命周期中，拦截器可以多次被调用，而过滤器只能在容器初始化时被调用一次。

6. 拦截器可以获取IoC容器中的各个Bean，而过滤器就不行，这点很重要，在拦截器里注入一个Service，可以调用业务逻辑。

在Spring中，AOP关注的是受IoC容器管理的对象，通常拦截的是对象是IoC管理中的Bean的业务方法，做一些业务前后的处理、日志异常捕捉等功能。

SpringMVC拦截器拦截的是用户请求，对每次用户请求都作出相关处理，作用类似于过滤器。
