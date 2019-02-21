---
title: 使用Java处理JSON数据格式
date: 2017-06-16 01:00:00
tags: [Java,JSON]
categories:
- Java
- JSON
---

本文代码：https://github.com/njZhuMin/Misc-codes/tree/master/Java/JavaJSON
- - -

# JSON的使用
## JSON数据格式
JSON是一种与开发语言无关的、轻量级的数据格式。JSONn之所以得到如此广泛的使用，是因为其易于人的阅读和编写，易于程序解析与生成的特点。

在标准JSON数据格式中，数据可以分为Object与Array两类。可以处理的基本数据类型有string、number、true、false和null。

其中Object对象使用花括号`{}`来标识，Object类型数据为键值对结构，其Key必须是string类型，value可以是任何基本类型或数据结构：
```js
{ key1: value, key2: value,  ... }
```

Array数据结构使用中括号`[]`标识，并使用逗号隔开其中的元素，其中value可以是任意的基本类型或数据结构：
```js
[ value1, value2, ... ]
```
标准的JSON格式中不支持任何形式的注释。

<!-- more -->

## 生成JSON格式
Java中有很多第三方的包来支持JSON格式数据的处理，这里使用最常用的`org.json`包来做demo。首先添加依赖：
```xml
<dependency>
    <groupId>org.json</groupId>
    <artifactId>json</artifactId>
    <version>20170516</version>
</dependency>
```

### 直接构造JSON对象
这里我们使用`JSONObject.put()`方法来构造一个JSON对象：
```java
private static void jSONObjectSample() {
    JSONObject laowang = new JSONObject();
    try {
        laowang.put("name", "隔壁老王");
        laowang.put("age", 25);
        laowang.put("birthday", "1990-01-01");
        laowang.put("school", "蓝翔");
        laowang.put("major", new String[] {"挖掘机", "计算机"});
        laowang.put("has_girlfriend", false);
        laowang.put("car", JSONObject.NULL);
        laowang.put("house", JSONObject.NULL);
        laowang.put("comment", "这是一个注释");
    } catch (JSONException e) {
        e.printStackTrace();
    }
    System.out.println(laowang.toString());
}
```
这里遇到一个小问题，一开始我们尝试put空对象：
```java
laowang.put("car", null);
```
编译器报了一个二义性错误，这里无法决定put函数的重载类型是`put(String, Collection<?>)`还是`put(String, Map<?, ?>)`。于是我们引入一个局部变量：
```java
Object nullObject = null;
laowang.put("car", nullObject);
```
成功通过了编译，但是在测试时，发现`<"car", null>`的键值对并没有出现在构造的JSON对象中。应该是put函数对空值做了过滤，看一眼`org.json.JSONObject`类的方法：
```java
/**
  * Put a key/value pair in the JSONObject. If the value is null, then the
  * key will be removed from the JSONObject if it is present.
  *
  * @param key
  *            A key string.
  * @param value
  *            An object which is the value. It should be of one of these
  *            types: Boolean, Double, Integer, JSONArray, JSONObject, Long,
  *            String, or the JSONObject.NULL object.
  * @return this.
  * @throws JSONException
  *             If the value is non-finite number or if the key is null.
  */
public JSONObject put(String key, Object value) throws JSONException {
    if (key == null) {
        throw new NullPointerException("Null key.");
    }
    if (value != null) {
        testValidity(value);
        this.map.put(key, value);
    } else {
        this.remove(key);
    }
    return this;
}
```
可以看到在put方法中，对于空的value，`this.remove(key)`，所以当然不会出现在构造的JSON对象中。那我们怎么把`<key, value=null>`放进JSON对象里呢？在该方法的参数注释中声明了，传入的value的数据类型必须为JSONArray、JSONObject、JSONObject.NULL和一些基本数据类型。我们找到`JSONObject.NULL`的定义：
```java
/**
  * JSONObject.NULL is equivalent to the value that JavaScript calls null,
  * whilst Java's null is equivalent to the value that JavaScript calls
  * undefined.
  */
private static final class Null {
    /**
      * There is only intended to be a single instance of the NULL object,
      * so the clone method returns itself.
      *
      * @return NULL.
      */
     @Override
     protected final Object clone() {
         return this;
     }
          @Override
     public boolean equals(Object object) {
         return object == null || object == this;
     }
     @Override
     public String toString() {
         return "null";
     }
}

public static final Object NULL = new Null();
```
可以看到，JSONObject类中定义了一个static final类`Null`，并且Override了这个类的`equals`方法，当传入的类是`null`值或一个`Null`类型的对象时，返回true。而`JSONObject.NULL`即是`Null`类的一个实例。

开发者也解释了这样做的目的是为了区分JS语言中的`null`和`undefined`两种类型。Java中的`null`对应JS中的`undefined`，此时编译器尚未给变量分配空间；而通过构造一个`Null`对象的实例`NULL`并传入，编译器会认为这是一个合法对象，其值是`null`，因此我们成功的向JSON对象中放入了空值。

### 使用Map创建JSON对象
第二种创建JSON对象的方法是传入一个Map对象。在`JSONObject`的类中我们可以看到以下定义：
```java
/**
  * The map where the JSONObject's properties are kept.
  */
  private final Map<String, Object> map;
```
即在`JSONObject`的核心就是将JSON格式保存在一个Map中。所以我们直接通过向`JSONObject`的构造函数中传入一个Map的方式，就可以创建一个JSON对象：
```java
private static void createJSONByMap() {
    Map<String, Object> laowang = new HashMap<String, Object>();
    laowang.put("name", "隔壁老王");
    laowang.put("age", 25);
    laowang.put("birthday", "1990-01-01");
    laowang.put("school", "蓝翔");
    laowang.put("major", new String[] {"挖掘机", "计算机"});
    laowang.put("has_girlfriend", false);
    laowang.put("car", JSONObject.NULL);
    laowang.put("house", JSONObject.NULL);
    laowang.put("comment", "这是一个注释");
    System.out.println(new JSONObject(laowang).toString());
}
```

### 使用JavaBean创建JSON对象
在日常开发中，我们更多的会使用JavaBean来创建JSON对象。我们先创建一个JavaBean：
```java
public class Person {
    private String name;
    private String birthday;
    private String school;
    private boolean has_girlfriend;
    private int age;
    private Object car;
    private Object house;
    private String[] major;
    private String comment;

    // ... getters and setters
}
```
然后我们实例化一个Bean对象并设置其属性，然后我们仍然通过构造函数直接传入我们的JavaBean对象，即可实现对应的JSON对象的创建：
```java
private static void createJSONByBean() {
    Person laowang = new Person();
    laowang.setName("隔壁老王");
    laowang.setAge(25);
    laowang.setBirthday("1990-01-01");
    laowang.setSchool("蓝翔");
    laowang.setMajor(new String[] {"挖掘机", "计算机"});
    laowang.setHas_girlfriend(false);
    laowang.setCar(null);
    laowang.setHouse(null);
    laowang.setComment("这是一个注释");
    System.out.println(new JSONObject(laowang));
}
```

## 解析JSON格式
为了简化文件读取，我们先在项目添加Apache Commons IO包的依赖：
```xml
<dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.5</version>
</dependency>
```
然后我们在当前路径下创建一个json文件：
```json
{
    "name": "隔壁老王",
    "age": 25,
    "birthday": "1990-01-01",
    "major": [
        "挖掘机",
        "计算机"
    ],
    "school": "蓝翔",
    "car": null,
    "house": null,
    "has_girlfriend": false,
    "comment": "这是一个注释"
}
```
然后我们读取这个文件，使用`JSONObject`中的一系列get方法来获取对应key的value。需要注意的时，JSONObject并不能直接返回原生的Java Array对象，需要从JSONArray数据类型进行转换：
```java
private static void JSONReadSample(String[] args) {
    File file = new File(ReadJSONSample.class.
            getResource("/laowang.json").getFile());
    try {
        String content = FileUtils.readFileToString(file, "utf-8");
        JSONObject jsonObject = new JSONObject(content);
        System.out.println("Name: " + jsonObject.getString("name"));
        System.out.println("Age: " + jsonObject.getInt("age"));
        System.out.println("Has_girlfriend: " + jsonObject.getBoolean("has_girlfriend"));

        JSONArray majorArray = jsonObject.getJSONArray("major");
        for(Object major : majorArray) {
            System.out.println("Major - " + major.toString());
        }
    } catch (IOException e) {
        e.printStackTrace();
    } catch (JSONException e) {
        e.printStackTrace();
    }
}
```

## 判断JSON是否为null
当我们从Web API或文件中读取JSON时，有时我们会遇到读取到的结果与期望不一致的情况（如取不到期望的key值）。我们可以用`isNull()`方法来判空：
```java
if(!jsonObject.isNull("name"))
    System.out.println("Name: " + jsonObject.getString("name"));
else
    throw new JSONException("Null key!");
```

# GSON工具包
[GSON](https://github.com/google/gson)是Google开源的JSON数据处理工具，以下是官方页面的原文，说明了GSON的设计理念：
> Gson is a Java library that can be used to convert Java Objects into their JSON representation. It can also be used to convert a JSON string to an equivalent Java object. Gson can work with arbitrary Java objects including pre-existing objects that you do not have source-code of.
>
>There are a few open-source projects that can convert Java objects to JSON. However, most of them require that you place Java annotations in your classes; something that you can not do if you do not have access to the source-code. Most also do not fully support the use of Java Generics. Gson considers both of these as very important design goals.

向项目中添加GSON包依赖：
```xml
<dependency>
    <groupId>com.google.code.gson</groupId>
    <artifactId>gson</artifactId>
    <version>2.8.1</version>
</dependency>
```

## GSON生成JSON数据
Gson提供了两种JSON生成方式，默认生成器会忽略null值以追求更高的压缩，而如果需要保留null值，可以使用`GsonBuilder().serializeNulls().create()`生成器来生成：
```java
private static void GSONCreateSample() {
    Person laowang = new Person();
    laowang.setName("隔壁老王");
    laowang.setAge(25);
    laowang.setBirthday("1990-01-01");
    laowang.setSchool("蓝翔");
    laowang.setMajor(new String[] {"挖掘机", "计算机"});
    laowang.setHas_girlfriend(false);
    laowang.setCar(null);
    laowang.setHouse(null);
    laowang.setComment("这是一个注释");

    // 压缩 null
    Gson gson1 = new Gson();
    System.out.println(gson1.toJson(laowang));
    // 保留 null
    Gson gson2 = new GsonBuilder().serializeNulls().create();
    System.out.println(gson2.toJson(laowang));
}
```

## 个性化的JSON构建
GSON支持在Bean类上使用`@SerializedName`注解来自定义JSON的key：
```java
public class Person {
    @SerializedName("NAME")
    private String name;
    // ...
}
```
另外，GSON支持自定义构造JSON生成器，例如打印美化的JSON数据：
```java
// pretty JSON
Gson gson3 = new GsonBuilder().setPrettyPrinting().create();
System.out.println(gson3.toJson(laowang));
```
更强大地，我们还可以在生成时使用回调函数来自定义名称转换策略：
```java
// FieldNaming Strategy
Gson gson4 = new GsonBuilder().setFieldNamingStrategy(neFieldNamingStrategy() {
    @Override
    public String translateName(Field f) {
        if(f.getName().equals("age"))
            return "AGE";
        return f.getName();
    }
}).create();
System.out.println(gson4.toJson(laowang));
```
有时我们不希望某些属性暴露给外界，但默认从JavaBean生成JSON时会将所有属性包括在内。GSON提供了对`transient`关键字的支持，被`transient`关键字标识的属性将不会参与到序列化，即JSON生成的过程中：
```java
public class Person {
    private transient String ignore;
    // ...
}

private static void GSONCreateSample() {
    Person laowang = new Person();
    laowang.setIgore("不要看见我");
    // ...
}
```
此时被我们标记为`transient`的属性ignore就不会被生成到JSON对象中去。

## 解析JSON到JavaBean
GSON甚至提供了直接将JSON格式解析并转换生成对应JavaBean的方法：
```java
private static void GSONReadSample(String[] args) {
    File file = new File(GSONReadSample.class.
            getResource("/laowang.json").getFile());
    try {
        String content = FileUtils.readFileToString(file, "utf-8");
        Gson gson = new Gson();
        Person laowang = gson.fromJson(content, Person.class);
        System.out.println(laowang);
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

## GSON中日期类型转换
GSON支持对日期格式类型的转换。首先我们在Bean中定义一个日期类型的属性：
```java
public class PersonWithBirthday {
    private Date birthday;
    // ...
}
```
然后我们使用`GSONBuilder`来创建一个生成器，自定义日期转换格式，就实现了对JSON对象中的日期格式的解析：
```java
private static void GSONReadSample(String[] args) {
    File file = new File(GSONReadSample.class.
            getResource("/laowang.json").getFile());
    try {
        String content = FileUtils.readFileToString(file, "utf-8");
        Gson gson2 = new GsonBuilder().setDateFormat("yyyy-MM-dd").create();
        PersonWithBirthday laowang_birthday = gson2.fromJson(content, PersonWithBirthday.class);
        System.out.println(laowang_birthday.getBirthday());
    } catch (IOException e) {
        e.printStackTrace();
    }
}
```

## 集合类解析
GSON还支持将JSON中的集合类自动转换成Java对应的集合类：
```java
public class PersonWithBirthday {
    private List<String> major;
    // ...
}

private static void GSONReadSample(String[] args) {
    File file = new File(GSONReadSample.class.
            getResource("/laowang.json").getFile());
    try {
        String content = FileUtils.readFileToString(file, "utf-8");
        Gson gson2 = new GsonBuilder().setDateFormat("yyyy-MM-dd").create();
        PersonWithBirthday laowang_birthday = gson2.fromJson(content, PersonWithBirthday.class);
        System.out.println(laowang_birthday.getBirthday());
        // 集合类的自动转换
        System.out.println(laowang_birthday.getMajor());
        System.out.println(laowang_birthday.getMajor().getClass());
    } catch (IOException e) {
        e.printStackTrace();
    }
}

// [挖掘机, 计算机]
// class java.util.ArrayList
```
可以看到，GSON自动将集合映射并匹配到了对应的ArrayList类，使我们可以直接将返回的结果作为Java原生集合类进行使用，十分方便。
