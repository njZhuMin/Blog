---
title: Ajax+Servlet实现搜索框智能提示
date: 2017-06-20 01:00:00
tags: [Servlet,Ajax]
categories:
- project
- AjaxSearch
---

本文项目代码：https://github.com/njZhuMin/AjaxSearch
- - -

# 需求分析
我们希望实现一个智能提示内容的搜索框。先来分析一下功能的流程：
1. 用户在搜索框输入关键字；
2. 浏览器将关键字异步发送给服务器；
3. 服务器结果处理，将相应的数据以JSON格式返回给客户端；
4. 客户端接收到服务器的响应数据，解析之后用JS操作DOM显示数据。

<!-- more -->

下面用一张图片来说明智能提示的流程：

{% asset_img flowchart.png flowchart %}

流程中的重点主要在于Ajax方式的数据交互，和使用JavaScript解析并动态展示返回的JSON数据。

# 前端页面
## 静态页面开发
先来写一个静态页面，只是负责展示一个简单的搜索框和一个搜索按钮：
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
    <head>
        <title>Search</title>
        <style type="text/css">
            #search {
                position: absolute;
                left: 50%;
                top: 50%;
                margin-left: -200px;
                margin-top: -50px;
            }
        </style>
    </head>
    <body>
        <div id="search">
            <!-- input box -->
            <input type="text" size="50" id="keyword" />
            <input type="button" value="Search" width="50px" />
        </div>
    </body>
</html>
```

## 创建XmlHttp对象
为了能够实时检测输入框的信息，我们给输入框绑定`onkeyup`事件：
```html
<script type="text/javascript">
    // get related content info
    function getMoreContents() {
        // get input text
        var content = document.getElementById("keyword");
        if(content.value == "")
            return;
        alert(content.value);
    }
</script>

<div id="search">
    <input type="text" size="50" id="keyword" onkeyup="getMoreContents()"/>
</div>
```
为了和服务器端实现异步通信，关键就是通过`xmlHttp`对象。这里考虑到不同浏览器的兼容性，我们将获取`xmlHttp`的方法单独提出来实现，接着使用`xmlHttp.open`方法创建一个`GET`方式的连接：
```html
<script type="text/javascript">
    var xmlHttp;
    // get related content info
    function getMoreContents() {
        // get input keyword
        var content = document.getElementById("keyword");
        if(content.value == "")
            return;
        // send keyword back to server via XmlHttp
        xmlHttp = getXmlHttp();
        var url = "search?keyword=" + encodeURI(content.value);
        // true: continue without waiting for response
        xmlHttp.open("GET", url, true);
    }

    function getXmlHttp() {
        var xmlHttp;
        // for most browser
        if(window.XMLHttpRequest)
            xmlHttp = new XMLHttpRequest();
        // IE compatibility
        if(Window.ActiveXObject) {
            xmlHttp = new ActiveXObject("Microsoft.XMLHTTP");
            if(!xmlHttp)
                xmlHttp = new ActiveXObject("Msxml2.XMLHTTP");
        }
        return xmlHttp;
    }
</script>
```

## 回调函数
接下来我们检测`xmlHttp`对象的状态并等待服务器的响应，当`xmlHttp`的状态值为4时，表示响应完成，此时出发绑定的回调函数：
```html
<script type="text/javascript">
function getMoreContents() {
    // get input keyword
    var content = document.getElementById("keyword");
    if(content.value == "")
        return;
    // send keyword back to server via XmlHttp
    xmlHttp = getXmlHttp();
    var url = "search?keyword=" + encodeURI(content.value);
    // true: continue without waiting for response
    xmlHttp.open("GET", url, true);
    // bind callback function, triggered by status change of xmlHttp
    // xmlHttp status: 0-4, 4: complete
    xmlHttp.onreadystatechange = callback;
    // params already in url with GET method
    xmlHttp.send(null);
}

function callback() {
    if(xmlHttp.readyState == 4) {
        if (xmlHttp.status == 200) {
            var result = xmlHttp.responseText;
            // parse JSON
            var json = eval("(" + result + ")");
            // display to DOM
            setContent(json);
        }
    }
}
</script>
```

## 展示数据到DOM
为了将从服务器返回得到的数据动态展示在输入框下方，我们先写一个预留的div：
```html
<div id="search">
    <!-- input box -->
    <input type="text" size="50" id="keyword" onkeyup="getMoreContents()"/>
    <input type="button" value="Search" width="50px" />
    <!-- display content below -->
    <div id="popDiv">
        <table id="content_table" bgcolor="#FFFAFA" border="0"
               cellspacing="0" cellpadding="0">
            <tbody id="content_table_body">
                <!-- some contents -->
            </tbody>
        </table>
    </div>
</div>
```
接下来我们来实现数据展示的函数：
```html
<script type="text/javascript">
// display content to DOM
function setContent(contents) {
    var size = contents.length;
    for(var i = 0; i < size; i++) {
        var nextNode = contents[i];
        var tr = document.createElement("tr");
        var td = document.createElement("td");
        td.setAttribute("border", "0");
        td.setAttribute("bgcolor", "#FFFAFA");
        td.onmouseover = function () {
            this.className = "mouseOver";
        };
        td.onmouseout = function () {
            this.className = "mouseOut";
        };
        td.onclick = function () {

        };

        var text = document.createTextNode(nextNode);
        td.appendChild(text);
        tr.appendChild(td);
        document.getElementById("content_table_body").appendChild(tr);
    }
}
</script>
```

# 后台开发
## Servlet的实现
我们之前定义了，当输入框触发了`mouseUp`事件是，我们就通过`xmlHttp`做一个`GET`请求：`search?keyword=input`。因此我们需要定义一个Servlet来拦截并响应这个url请求。实现的方式也很简单，我们直接继承`HttpServlet`类并override其`doGet`方法即可：
```java
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    req.setCharacterEncoding("utf-8");
    resp.setCharacterEncoding("utf-8");
    // get keyword
    String keyword = req.getParameter("keyword");
    List<String> listData = getData(keyword);
    for(String d : listData)
        System.out.println(d);
    // result list to JSON
    Gson gson = new Gson();
    String json = gson.toJson(listData);
    resp.getWriter().write(json);
}

private List<String> getData(String keyword) {
    List<String> list = new ArrayList<String>();
    for(String data : datas)
        if(data.contains(keyword))
            list.add(data);
    return list;
}
```

## 细节问题的处理
现在我们已经可以将数据展现在div层了，但是输出结果每次都在递增，这是因为我们每接受一次结果就添加到了结果列表的后面。因此我们要实现一个函数，在每次`setContent`前先调用清空原有的结果：
```js
function clearContent() {
    var contentTableBody = document.getElementById("content_table_body");
    var size = contentTableBody.childNodes.length;
    for(var i = size - 1; i >= 0; i--)
        contentTableBody.removeChild(contentTableBody.childNodes[i]);
}
```
然后我们继续来研究细节问题。当我们输入错误，删除掉之前的输入，或是点击页面其他空白部位时，我们期望提示的内容应当跟着改变或消失。

另外，当我们鼠标移动到当前条目时，希望被选择的条目颜色有伴随的高亮变化。当单击某一条目时，能够将当前条目自动填充到输入框内。
```js
// display content to DOM
function setContent(contents) {
    // clear content first
    clearContent();
    // set div position
    setLocation();
    var size = contents.length;

    for(var i = 0; i < size; i++) {
        var nextNode = contents[i];
        var tr = document.createElement("tr");
        var td = document.createElement("td");
        td.setAttribute("border", "0");
        td.setAttribute("bgcolor", "#FFFAFA");
        td.onmouseover = function () {
            this.className = "mouseOver";
        };
        td.onmouseout = function () {
            this.className = "mouseOut";
        };
        td.onmousedown = function () {
            document.getElementById("keyword").value = this.innerText;
        };

        var text = document.createTextNode(nextNode);
        td.appendChild(text);
        tr.appendChild(td);
        document.getElementById("content_table_body").appendChild(tr);
    }
}

function clearContent() {
    var contentTableBody = document.getElementById("content_table_body");
    var size = contentTableBody.childNodes.length;
    for(var i = size - 1; i >= 0; i--)
        contentTableBody.removeChild(contentTableBody.childNodes[i]);
    document.getElementById("popDiv").style.border = "none";
}

// when input box loses focus
function keywordBlur() {
    clearContent();
}

function setLocation() {
    var content = document.getElementById("keyword");
    // width of input box
    var width = content.offsetWidth;
    var left = content["offsetLeft"];
    var top = content["offsetTop"] + content.offsetHeight;
    // get display div
    var popDiv = document.getElementById("popDiv");
    popDiv.style.border = "black 1px solid";
    popDiv.style.left = left + "px";
    popDiv.style.top = top + "px";
    popDiv.style.width = width + "px";
    document.getElementById("content_table").style.width = width + "px";
}
```

还有一个小细节，就是当我们不是删除关键字，而是当前关键字没有对应的提示信息，即返回的JSON为空时，也需要隐藏div层的边框：
```js
function setContent(contents) {
    // clear content first
    clearContent();
    // set div position
    setLocation();
    var size = contents.length;
    if(size == 0) {
        clearContent();
        return;
    }
    // ...
}
```

# IDEA“智能”的插曲
在整个项目开发的过程中，因为IDEA的“智能化”发生了两个小插曲，特此记录一下。

第一个问题发生在后端测试时，从前端页面发送的关键字已经成功接收到，并且提示的JSON信息也在后台log测试成功。但将JSON数据返回发送给前台时却失败了。现在Chrome中F12调试一下，发现报了405的错误，想到可能是Servlet的问题，于是去检查Servlet。一行行的测试都没什么问题，为什么前端就是405错误呢？

后来终于发现，是因为IDEA在我定义了SearchSerlvet继承HttpServlet并override `doGet`方法的时候，自动添加了一句`super.doGet(req, resp)`，这样我们自定义的`doGet`方法自然就失效了，因为request已经被错误的放行了。删掉这一句再测试就OK了，前端可以正确的获取JSON结果了。

第二个问题是我在向Maven中添加了`<packaging>war</packaging>`之后，突然所有页面都404了。研究了半天，还做了`mvn clean`，还是找不到页面。后来发现这个错误是由于IDEA的项目结构变化导致的。

当我们一开始在IDEA向项目中添加Web框架支持时，IDEA会自动生成`Web`文件夹和`web.xml`等相关Web资源，并且会自动在项目中配置`Web`文件夹为Web资源根目录。而在我加了`<packaging>war</packaging>`之后，Maven接管了整个项目的构建，在`src/main/`下自动创建了`webapp`的目录作为Web资源的根目录，并且会将项目配置中的Web资源目录自动指向`webapp`。而这个新生成的目录是空的，自然找不到资源。因此我们只要将原来的`Web`文件夹下的所有文件移动到`webapp`中，rebuild一下项目，就成功解决了这个莫名其妙的404错误。

# 总结
至此我们已经利用Ajax与Servlet基本实现了一个简易的智能提示的搜索框的功能。本文中项目的前段代码使用了最原始的写法，通过`xmlHttp`体现整个Ajax请求的过程，是为了更好的展示其原理。包括js和css的设置也是采用的最直接却稍显繁琐的方式，目的也是更好的体验何时触发何种js方法的设计过程。

大家当然可以采用更简洁先进的jQuery写法来重构前端代码，但我觉得在学习的初级阶段，优秀的工具毕替我们简化了代码书写，却也黑盒掉了大量的实现细节。不先搞懂其中的原理就拿来主义的大量使用，总会埋下些隐患。我们对待一些高级特性的工具、框架等技术，还是应虚心的、耐心的知其然并且知其所以然，才能让这些技术更好地为我们所用。
