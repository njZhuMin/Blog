---
title: MyBatis-3快速入门
date: 2016-04-25 20:31:16
tags: [mybatis,mapping,ORM]
categories:
- framework
- MyBatis
---

# 准备开发环境
## 创建测试项目

MyBatis 是支持定制化 SQL、存储过程以及高级映射的优秀的持久层框架。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以对配置和原生Map使用简单的 XML 或注解，将接口和 Java 的 POJOs(Plain Old Java Objects,普通的 Java对象)映射成数据库中的记录。

这里首先创建一个Java项目`MyBatis`，普通的或者JavaWeb均可。

<!-- more -->

## 添加相应的jar包

首先，在[MyBatis官网](http://www.mybatis.org/mybatis-3/zh/getting-started.html)获取jar包：[mybatis-x.x.x.jar](https://github.com/mybatis/mybatis-3/releases)。
这里以MySQL为例，因此前往MySQL官网下载[DBC Driver for MySQL (Connector/J)](http://dev.mysql.com/downloads/connector/j/)。

将下载得到的`mybatis-x.x.x.jar`与`mysql-connector-java-x.x.x-bin.jar`复制到项目目录下的lib文件夹。目录结构如下（以IDEA为例）：
> MyBatis
> &nbsp;&nbsp;&nbsp;|--- .idea
> &nbsp;&nbsp;&nbsp;|--- lib
> &nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp;|--- mybatis-x.x.x.jar
> &nbsp;&nbsp;&nbsp;|&nbsp;&nbsp;&nbsp;&nbsp;|--- mysql-connector-java-x.x.x-bin.jar
> &nbsp;&nbsp;&nbsp;|--- src
> &nbsp;&nbsp;&nbsp;|--- MyBatis.iml

## 创建数据库与表
使用`mysql -u {$user} -p`创建数据库与表：
```sql
CREATE DATABASE mybatis;
USE mybatis;
CREATE TABLE users (
    id int PRIMARY KEY AUTO_INCREMENT,
    name, varchar(20),
    age int);
INSERT INTO users(NAME, age) VALUES('TEST1', 20);
INSERT INTO users(NAME, age) VALUES('TEST2', 22);
```
至此，前期的开发环境准备工作全部完成。

# 使用MyBatis查询表中的数据
## 添加Mybatis的配置文件config.xml
在`src`目录下创建一个`config.xml`文件，内容如下：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value="minmin95420"/>
            </dataSource>
        </environment>
    </environments>
</configuration>
```

## 定义表所对应的实体类User.java
```java
package com.sunnywr.domain;

public class User {
    private int id;
    private String name;
    private int age;

    public int getId() {
        return id;
    }
    public void setId(int id) {
        this.id = id;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User [id=" + id + ", name=" + name + ", age=" + age + "]";
    }
}
```

## 定义映射文件userMapper.xml
创建包`com.sunnywr.mapping`，存放sql映射文件userMapper.xml，内容如下：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- 为这个mapper指定一个唯一的namespace
    namespace的值习惯上设置成包名+sql映射文件名，这样就能够保证namespace的值是唯一的 -->
<mapper namespace="com.sunnywr.mapping.userMapper">
    <!-- 在select标签中编写查询的SQL语句， 设置select标签的id属性为getUser
    id属性值必须是唯一的，不能够重复
    使用parameterType属性指明查询时使用的参数类型，resultType属性指明查询返回的结果集类型 -->
    <!-- 根据id查询得到一个user对象 -->
    <select id="getUser" parameterType="int"
            resultType="com.sunnywr.domain.User">
        select * from users where id=#{id}
    </select>
</mapper>
````

## 在conf.xml文件中注册userMapper.xml文件
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis"/>
                <property name="username" value="root"/>
                <property name="password" value="minmin95420"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <!-- 注册userMapper.xml文件 -->
        <mapper resource="com/sunnywr/mapping/userMapper.xml"/>
    </mappers>
</configuration>
```

## 编写测试代码
创建一个Test类，编写测试代码，执行定义的select语句。
```java
package com.sunnywr.test;

import java.io.IOException;
import java.io.InputStream;
import com.sunnywr.domain.User;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

public class Test {
    public static void main(String[] args) throws IOException {
        //mybatis的配置文件
        String resource = "config.xml";
        //使用类加载器加载mybatis的配置文件（它也加载关联的映射文件）
        InputStream is = Test.class.getClassLoader().getResourceAsStream(resource);
        //构建sqlSession的工厂
        SqlSessionFactory sessionFactory = new SqlSessionFactoryBuilder().build(is);
        //使用MyBatis提供的Resources类加载mybatis的配置文件（它也加载关联的映射文件）
        //Reader reader = Resources.getResourceAsReader(resource);
        //构建sqlSession的工厂
        //SqlSessionFactory sessionFactory = new SqlSessionFactoryBuilder().build(reader);
        //创建能执行映射文件中sql的sqlSession
        SqlSession session = sessionFactory.openSession();
        /**
         * 映射sql的标识字符串，
         * me.gacl.mapping.userMapper是userMapper.xml文件中mapper标签的namespace属性的值，
         * getUser是select标签的id属性值，通过select标签的id属性值就可以找到要执行的SQL
         */
        String statement = "com.sunnywr.mapping.userMapper.getUser";    //映射sql的标识字符串
        //执行查询返回一个唯一user对象的sql
        User user1 = session.selectOne(statement, 1);
        User user2 = session.selectOne(statement, 2);
        System.out.println(user1);
        System.out.println(user2);
    }
}
```

测试结果：
> User [id=1, name=TEST1, age=20]
> User [id=2, name=TEST2, age=22]
