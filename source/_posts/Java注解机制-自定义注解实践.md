---
title: Java注解机制-自定义注解实践
date: 2016-05-10 10:46:13
tags: [Java,annotation]
categories:
- Java
- annotation
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

> 测试代码：
> https://github.com/njZhuMin/JavaDemo/tree/master/annotation/AnnotationDemo

- - -

# 需求
1. 有一张用户表`user`，字段包括用户ID，用户名，昵称，年龄，性别，所在城市，邮箱，手机号；

2. 方便对每个字段或字段的组合条件进行检索，并打印出SQL语句；

3. 使用方式简单，通过注解实现。

## Filter.java
```java
public class Filter {
    private int id;
    private String userName;
    private String nickName;
    private int age;
    private String city;
    private String email;
    private String mobile;

	//getters and setters ommitted here
	...
```

<!-- more -->

## Test.java
```java
package com.sunnywr;

public class Test {
    public static void main(String[] args) {
        Filter f1 = new Filter();
        f1.setId(10);

        Filter f2 = new Filter();
        f2.setUserName("Lucy");

        Filter f3 = new Filter();
        f3.setEmail("liu@sina.com, zh@163.com, 77777@qq.com");

        String sql1 = query(f1);
        String sql2 = query(f2);
        String sql3 = query(f3);

        System.out.println(sql1);
        System.out.println(sql2);
        System.out.println(sql3);
    }

    private static String query(Filter filter) {
        return null;
    }
}
```

# 需求分析
首先我们进行分析，需求中要对数据库进行查询（给出SQL语句），那么我们首先应该考虑`代码如何与数据库进行映射`的问题。因为对数据库做SQL查询需要表名与字段名，所以我们首先应对Filter这个Bean类添加表名与字段的注解。
```java
@Table("user")
public class Filter {
    @Column("id")
    private int id;

    @Column("user_name")
    private String userName;

    @Column("nick_name")
    private String nickName;

    @Column("age")
    private int age;

    @Column("city")
    private String city;

    @Column("email")
    private String email;

    @Column("mobile")
    private String mobile;

    public int getId() {
        return id;
    }

	//getters and setters ommitted here
	...
```

# 定义注解
```java
package com.sunnywr;
import java.lang.annotation.*;

@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Table {
    String value();
}
```

```java
package com.sunnywr;
import java.lang.annotation.*;

@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
    String value();
}
```

# 解析注解初步
```java
package com.sunnywr;
import java.lang.reflect.Field;
import java.lang.reflect.Method;

public class Test {
	//main method ommitted
	...

	private static String query(Filter filter) {
        StringBuilder sb = new StringBuilder();
        //获取Class
        Class c = filter.getClass();

        //获取table name
        if(!c.isAnnotationPresent(Table.class))
            return null;
        Table t = (Table)c.getAnnotation(Table.class);
        String tableName = t.value();
        sb.append("SELECT * FROM ").append(tableName).append(" WHERE 1=1");

        //遍历所有字段
        Field[] fields = c.getDeclaredFields();
        for(Field field : fields) {
            //处理每个字段的SQL语句
            //1. 获取字段名
            if(c.isAnnotationPresent(Column.class))
                continue;
            Column column = field.getAnnotation(Column.class);
            String columnName = column.value();

            //2. 获取字段值
            String fieldName = field.getName();
            String getMethodName = "get" + fieldName.substring(0,1).toUpperCase()
                    + fieldName.substring(1);
            String fieldValue = null;
            try {
                Method getMethod = c.getMethod(getMethodName);
                fieldValue = (String)getMethod.invoke(filter);
            } catch (Exception e) {
                e.printStackTrace();
            }

            //3. 组装SQL语句
            sb.append(" and ").append(fieldName).append("=").append(fieldValue);
        }
        return sb.toString();
    }
}
```

# 问题和解决
## 改进1：初步debug
这样我们就基本完成了注解的简易版本。当我们运行测试的时候发现还是有一些问题的。
> 1. 因为并不是所有的field都是String类型的，`String fieldValue = null;`的语句报类型转换错误。

> 2. 打印出的条件中出现`column=null`和`column=0`等条件，应该去除这些条件。

因此我们针对性的改进代码：
> 1. 将fieldValue类型抽象为`Object`，取消对String的强制类型转换。

> 2. 在组装SQL语句时增加对条件的判断，筛除`null`和`int=0`的条件。

输出如下：
```sql
SELECT * FROM user WHERE 1=1 and id=10
SELECT * FROM user WHERE 1=1 and userName=Lucy
SELECT * FROM user WHERE 1=1 and email=liu@sina.com, zh@163.com, 77777@qq.com
```

## 改进2：字符串两侧引号问题
```java
//3. 组装SQL语句
//处理null和0
if(fieldValue == null ||
		(fieldValue instanceof Integer && (Integer)fieldValue == 0))
    continue;
sb.append(" and ").append(fieldName);
//处理String引号
if(fieldValue instanceof String) {
	sb.append("=").append("'").append(fieldValue).append("'");
}
else if(fieldValue instanceof Integer) {
    sb.append("=").append(fieldValue);
}
```

## 改进3：多条件查询实现
```java
//3. 组装SQL语句
//处理null和0
if(fieldValue == null ||
        (fieldValue instanceof Integer && (Integer)fieldValue == 0))
    continue;
sb.append(" AND ").append(fieldName);
//处理String引号
if(fieldValue instanceof String) {
    //处理多个查询条件
    if(((String) fieldValue).contains(",")) {
        String[] values = ((String) fieldValue).split(",");
        sb.append(" IN(");
        for(String v : values) {
            sb.append("'").append(v).append("'").append(",");
        }
        //去掉最后一个逗号
        sb.deleteCharAt(sb.length()-1);
        sb.append(")");
    } else {
        sb.append("=").append("'").append(fieldValue).append("'");
    }
}
else if(fieldValue instanceof Integer) {
    sb.append("=").append(fieldValue);
}
```

# 总结
## 注解的好处
现在如果我们需要对另一张表`department`生成SQL语句，我们的注解类是不是仍然正确呢？

```java
@Table("department")
public class Filter2 {
    @Column("id")
    private int id;

    @Column("name")
    private String name;

    @Column("leader")
    private String leader;

    @Column("amount")
    private int amount;

	//getters and setters ommitted
	...
```

```java
Filter2 filter2 = new Filter2();
filter2.setAmount(10);
filter2.setName("技术部");
System.out.println(query(filter2));
```

可以看到，即使对于不同的表，使用注解仍然可以返回出正确的SQL查询语句。
