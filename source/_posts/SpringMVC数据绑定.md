---
title: SpringMVC数据绑定
date: 2016-06-16 09:31:16
tags: [SpringMVC,binding]
categories:
- framework
- SpringMVC
---

> SpringMVC系列文章：
> http://zhum.in/blog/categories/frame/SpringMVC/

> 示例代码：
> https://github.com/njZhuMin/SpringMVCDemo

- - -
# 基本类型与包装类型
1. 基本类型与包装类型
以年龄age属性为例，如果age为int类型，那么age不能为空，取值范围也必须符合合int的范围。如果age不是整形，如abc，服务器会报一个400的错误；而如果age为空，就会报一个500错误。

而包装类型就允许空值的存在。因此开发过程中，对于可能为空的数据应使用包装类型。同时，还可以使用`@RequestParam`注解来配置某个参数是否是必须的。

<!-- more -->

2. SpringMVC支持绑定的数据类型：基本类型、包装类型、String对象类型

```java
// http://127.0.0.1:8080/baseType.do?age=10
@RequestMapping("baseType.do")
@ResponseBody
public String baseType(int age) {
    return "age: " + age;
}

// http://127.0.0.1:8080/baseType2.do?age=10
@RequestMapping("baseType2.do")
@ResponseBody
public String baseType2(Integer age) {
    return "age: " + age;
}

// http://127.0.0.1:8080/array.do?name=Tom&name=Lucy&name=Jim
@RequestMapping("array.do")
@ResponseBody
public String array(String[] name) {
    StringBuilder sbf = new StringBuilder();
    for(String item : name) {
        sbf.append(item).append(" ");
    }
    return sbf.toString();
}
```

# 简单对象、多层级对象及同属性对象
> 使用`@InitBinder`注解进行namespace的绑定

```java
// http://127.0.0.1:8080/object.do?name=Tom&age=10&contactInfo.phone=10086
// http://127.0.0.1:8080/object.do?user.name=Tom&admin.name=Lucy&user.age=10
@RequestMapping("object.do")
@ResponseBody
public String object(User user, Admin admin) {
    return user.toString() + " " + admin.toString();
}

@InitBinder("user")
public void initUser(WebDataBinder binder) {
    binder.setFieldDefaultPrefix("user.");
}

// http://127.0.0.1:8080/object.do?user.name=Tom&admin.name=Lucy&user.age=10
@InitBinder("admin")
public void initAdmin(WebDataBinder binder) {
    binder.setFieldDefaultPrefix("admin.");
}
```

# 集合类型的绑定
## List
1. springMVC不支持List类型数据的直接绑定，需要包装成Object
2. 如果传递参数中的下标是不连续的，中间会被赋空值，需要注意

```java
// http://127.0.0.1:8080/list.do?users[0].name=Tom&users[1].name=Lucy
@RequestMapping("list.do")
@ResponseBody
public String list(UserListForm userListForm) {
     return "size=" + userListForm.getUsers().size() + " "
        	+ userListForm.toString();
}
```

## Set
1. Set类型的数据进行绑定时必须进行初始化，而List则不用

2. Set一般用于重复判断或排除重复

3. Set进行数据绑定与List非常相似，但必须初始化，并指定size空间，否则会抛出异常。形如List的序号跨界如果超出了size范围也是不允许的。

4. 要使用Set的排重功能必须在对象中覆写hashcode和equals方法。SpringMVC对Set支持并不太好，初始化进行排重时会导致size变小，致使无法接受更多的数据而抛出异常，所以开发时一般优先使用List。

```java
// 初始化Set的size
public UserSetForm() {
    users = new LinkedHashSet<User>();
    users.add(new User());
    users.add(new User());
}
```
对于未初始化或下标越界的参数，服务器会报500异常。具体见`org.springframework.beans.AbstractNestablePropertyAccessor`：
```java
else if (value instanceof Set) {
    // Apply index to Iterator in case of a Set.
    Set<Object> set = (Set<Object>) value;
    int index = Integer.parseInt(key);
    if (index < 0 || index >= set.size()) {
        throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                "Cannot get element with index " + index + " from Set of size " +
                        set.size() + ", accessed using property path '" + propertyName + "'");
    }
    Iterator<Object> it = set.iterator();
    for (int j = 0; it.hasNext(); j++) {
        Object elem = it.next();
        if (j == index) {
            value = elem;
            break;
        }
    }
}
```

另外，为了实现去重，我们需要复写User类的`hashCode()`与`Equals()`方法：
```java
@Override
public int hashCode() {
    return Objects.hash(name, age);
}

@Override
public boolean equals(Object obj) {
    if(this == obj)
        return true;
    if(obj == null || this.getClass() != obj.getClass())
        return false;
    User user = (User)obj;
    return Objects.equals(name, user.name) && Objects.equals(age, user.age);
}
```

但是这样，当我们传入参数`users[0].name=Tom&users[1].name=Tom`时，初始化中的Set的size()又变为了1，又出现了越界的异常。因此我们说SpringMVC对于Set数据类型的绑定支持不够好。

## Map
```java
// http://127.0.0.1:8080/map.do?users['X'].name=Tom&users['X'].age=10&users['Y'].name=Lucy
@RequestMapping("map.do")
@ResponseBody
public String map(UserMapForm userMapForm) {
    return userMapForm.toString();
}
```

# Json与XML
```java
// {
//     "name":"Jim",
//         "age":16,
//         "contactInfo":{
//             "address":"Beijing",
//             "phone":"10086"
//         }
// }
@RequestMapping("json.do")
@ResponseBody
public String json(@RequestBody User user) {
    return user.toString();
}

// <?xml version="1.0" encoding="UTF-8" ?>
// <admin>
//     <name>Jim</name>
//     <age>16</age>
// </admin>
@RequestMapping("xml.do")
@ResponseBody
public String xml(@RequestBody Admin admin) {
    return admin.toString();
}
```

# 自定义类型转换器
## PropertyEditor
`PropertyEditor`是Java中的一个接口类，有一个实现类`PropertyEditorSupport`。其中比较重要的方法有`setValue`、`getValue`和`setAsText`方法。

另外还有`Formatter`和`Converter`类实现格式化功能。以`PropertyEditor`为例，我们一般不直接实现`PropertyEditor`接口，而是通过继承`PropertyEditorSupport`类来进行扩展。

```java
public class MyPropertyEditor extends PropertyEditorSupport {
    @Override
    public void setAsText(String text) throws IllegalArgumentException {
        User user = new User();
        String[] textArray = text.split(",");
        user.setName(textArray[0]);
        user.setAge(Integer.parseInt(textArray[1]));
        this.setValue(user);
    }

    public static void main(String[] args) {
        MyPropertyEditor editor = new MyPropertyEditor();
        editor.setAsText("Tom,22");
        System.out.println(editor.getValue());
    }
}
```

## 自定义类型的绑定
以Date类型为例，首先看到SpringMVC中的`CustomDateEditor`类，其中注释：
```java
/ * <p>In web MVC code, this editor will typically be registered with
 * {@code binder.registerCustomEditor}.
 *
 *  org.springframework.validation.DataBinder#registerCustomEditor
 */
public class CustomDateEditor extends PropertyEditorSupport {
}
```
继续去看binder的实现类`WebDataBinder`，发现其中没有`registerCustomEditor`的实现。查看它的父类`DataBinder`：
```java
@Override
public void registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor) {
    registerCustomEditor(requiredType, null, propertyEditor);
}

@Override
public void registerCustomEditor(Class<?> requiredType, String propertyPath, PropertyEditor propertyEditor) {
    if (requiredType == null && propertyPath == null) {
        throw new IllegalArgumentException("Either requiredType or propertyPath is required");
    }
    if (propertyPath != null) {
        if (this.customEditorsForPath == null) {
            this.customEditorsForPath = new LinkedHashMap<String, CustomEditorHolder>(16);
        }
        this.customEditorsForPath.put(propertyPath, new CustomEditorHolder(propertyEditor, requiredType));
    }
    else {
        if (this.customEditors == null) {
            this.customEditors = new LinkedHashMap<Class<?>, PropertyEditor>(16);
        }
        this.customEditors.put(requiredType, propertyEditor);
        this.customEditorCache = null;
    }
}
```
跟踪`getPropertyEditorRegistry`方法的返回值是`PropertyEditorRegistry`类，发现这个类是一个接口，其中一个实现类`PropertyEditorRegistrySupport`。看到有两个成员变量`defaultEditors`和`customEditors`。

`defaultEditors`是初始化时默认创建的，而`customEditors`则需要我们手动创建。查看它的创建方法，首先定义了一个64长度的HashMap，然后添加了一些map值：
```java
private void createDefaultEditors() {
    this.defaultEditors = new HashMap<Class<?>, PropertyEditor>(64);

    // Simple editors, without parameterization capabilities.
    // The JDK does not contain a default editor for any of these target types.
    this.defaultEditors.put(Charset.class, new CharsetEditor());
    this.defaultEditors.put(Class.class, new ClassEditor());
    this.defaultEditors.put(Class[].class, new ClassArrayEditor());
    this.defaultEditors.put(Currency.class, new CurrencyEditor());
    this.defaultEditors.put(File.class, new FileEditor());
    // ...
    if (zoneIdClass != null) {
        this.defaultEditors.put(zoneIdClass, new ZoneIdEditor());
    }
}
```

那么我们如何在SpringMVC中使用`CustomDateEditor`进行数据类型转换呢：
```java
@RequestMapping("date1.do")
@ResponseBody
public String date1(Date date1) {
    return date1.toString();
}

@InitBinder("date1")
public void initDate1(WebDataBinder binder) {
    binder.registerCustomEditor(Date.class, new CustomDateEditor(
            new SimpleDateFormat("yyyy-MM-dd"),true));
}
```

## Formatter与Converter
如果我们需要实现全局的数据类型转换呢？
首先实现`org.springframework.format.Formatter`接口：
```java
public class MyDateFormatter implements Formatter<Date> {
    public Date parse(String text, Locale locale) throws ParseException {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        return sdf.parse(text);
    }

    public String print(Date object, Locale locale) {
        return null;
    }
}
```

在配置文件中注册Formatter：
```xml
<!-- 扩充了注解驱动，可以将请求参数绑定到控制器参数 -->
<mvc:annotation-driven conversion-service="myDateFormatter"/>

<bean id="myDateFormatter" class="org.springframework.format.support.FormattingConversionServiceFactoryBean">
    <property name="formatters">
        <set>
            <bean class="com.zhum.inmon.MyDateFormatter"></bean>
        </set>
    </property>
</bean>
```

使用Formatter：
```java
// http://127.0.0.1:8080/date2.do?date2=2016-08-18
@RequestMapping("date1.do")
@ResponseBody
public String date1(Date date1) {
    return date1.toString();
}
```

`Converter`的实现与`Formatter`相似，但支持使用自定义类型作为parser。

# RESTful
REST，即Resource Representation State Transfer，资源表现层状态转化。如果一个架构符合REST原则，就成它为RESTful架构。

## 表现层
以`http://localhost:8080/book`为例，其中的book只代表资源的实体，而要资源怎样的表现形式则交给Content-Type进行指定，不在URL进行指定（区别于以前在URL请求中使用book.do或book.html等）：
```java
// http://127.0.0.1:8080/book
@RequestMapping(value = "/book", method = RequestMethod.GET)
@ResponseBody
public String book(HttpServletRequest request) {
    String contentType = request.getContentType();
    if(contentType == null) {
        return "book.default";
    } else if(contentType.equals("txt")) {
        return "book.txt";
    } else if(contentType.equals("html")) {
        return "book.html";
    }
    return "book.default";
}
```

## 状态转化
客户端与服务器通信，要使服务器的状态发生变化，常用的有以下四种HTTP动词：
- GET：获取资源
- POST：创建资源（不具有幂等性）
- PUT：创建（更新）资源
- DELETE：删除资源

幂等性：每次HTTP请求相同的参数、相同的URI，产生的结果是相同的。

```java
// http://127.0.0.1:8080/subject/123456
@RequestMapping(value = "/subject/{subjectId}", method = RequestMethod.GET)
@ResponseBody
public String subjectGet(@PathVariable("subjectId") String subjectId) {
    return "this is a get method. subjectId: " + subjectId;
}

@RequestMapping(value = "/subject/{subjectId}", method = RequestMethod.POST)
@ResponseBody
public String subjectPost(@PathVariable("subjectId") String subjectId) {
    return "this is a post method. subjectId: " + subjectId;
}

@RequestMapping(value = "/subject/{subjectId}", method = RequestMethod.DELETE)
@ResponseBody
public String subjectDelete(@PathVariable("subjectId") String subjectId) {
    return "this is a delete method. subjectId: " + subjectId;
}

@RequestMapping(value = "/subject/{subjectId}", method = RequestMethod.PUT)
@ResponseBody
public String subjectPut(@PathVariable("subjectId") String subjectId) {
    return "this is a put method. subjectId: " + subjectId;
}
```



