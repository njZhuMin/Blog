---
title: SpringData使用简介
date: 2017-6-20 01:00:00
tags: [SpringData]
categories:
- framework
- SpringData
---

# Spring Data概述
Spring Data是Spring系列中的子项目，由Rod Johnso在2010年正式发布。Spring Data项目的主旨是提供与Spring模型一致性的数据库开发体验，简化数据库的访问（涵盖关系型与非关系型数据库的支持）。

Spring Data项目覆盖众多应用场景：
- Spring Data Jpa: 统一的Jpa数据访问接口
- Spring Data MongoDB: 对底层MongoDB快速访问
- Spring Data Redis: 提供Redis快速访问接口
- Spring Data Solr: 便捷的Solr数据访问
- ...

<!-- more -->

# JDBC访问数据库
传统的JDBC数据库访问方式，通过Connection对象建立数据库连接，使用Statement构建SQL语句，通过ResultSet获取返回的查询结果。

我们先来展示一个标准的JDBC流程：
```java
public class Student {

    private int id;
    private String name;
    private int age;
    // ... constructors, toString, getters and setters
}
```
```java

/**
 * JDBC工具类
 */
public class JDBCUtil {

    /**
     * 获取Connection对象
     * @return
     */
    public static Connection getConnection() throws Exception {

        InputStream inputStream = JDBCUtil.class.getClassLoader().getResourceAsStream("db.properties");
        Properties properties = new Properties();
        properties.load(inputStream);

        String driverClass = properties.getProperty("jdbc.driverClass");
        String url = properties.getProperty("jdbc.url");
        String user = properties.getProperty("jdbc.user");
        String password = properties.getProperty("jdbc.password");

        Class.forName(driverClass);
        Connection connection = DriverManager.getConnection(url, user, password);
        return connection;
    }

    /**
     * 释放数据库相关资源
     * @param resultSet
     * @param statement
     * @param connection
     */
    public static void releaseResources(ResultSet resultSet, Statement statement,
                                        Connection connection) {
        if(resultSet != null) {
            try {
                resultSet.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
        if(statement != null) {
            try {
                statement.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
        if(connection != null) {
            try {
                connection.close();
            } catch (SQLException e) {
                e.printStackTrace();
            }
        }
    }
}
```
```java
public interface StudentDao {

    /**
     * 查询所有学生信息
     */
    public List<Student> queryList();

    /**
     * 向数据库插入一条学生记录
     * @param student
     */
    public void insertOne(Student student);
}

public class StudentDaoImpl implements StudentDao {

    @Override
    public List<Student> queryList() {

        Connection connection = null;
        PreparedStatement preparedStatement = null;
        ResultSet resultSet = null;
        String sql = "SELECT id, name, age FROM student";

        List<Student> students = new ArrayList<>();
        try {
            connection = JDBCUtil.getConnection();
            preparedStatement = connection.prepareStatement(sql);
            resultSet = preparedStatement.executeQuery();

            while(resultSet.next()) {
                int id = resultSet.getInt("id");
                String name = resultSet.getString("name");
                int age = resultSet.getInt("age");
                students.add(new Student(id, name, age));
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            JDBCUtil.releaseResources(resultSet, preparedStatement, connection);
        }
        return students;
    }

    @Override
    public void insertOne(Student student) {
        Connection connection = null;
        PreparedStatement preparedStatement = null;
        ResultSet resultSet = null;
        String sql = "INSERT INTO student(name, age) VALUES(?, ?)";

        try {
            connection = JDBCUtil.getConnection();
            preparedStatement = connection.prepareStatement(sql);
            preparedStatement.setString(1, student.getName());
            preparedStatement.setInt(2, student.getAge());
            preparedStatement.executeUpdate();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            JDBCUtil.releaseResources(resultSet, preparedStatement, connection);
        }
    }
}
```
可以看到在Dao层的实现类中，有大量重复冗余的代码。这样的开发效率显然是极其低下的。

# Spring JdbcTemplate
Spring框架为我们提供了Spring JdbcTemplate的方法，大大简化了数据库操作的开发过程。

首先我们添加Spring JDBC工具包的依赖：
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>4.3.9.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>4.3.9.RELEASE</version>
</dependency>
```
配置Spring-Context的Beans注入：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="username" value="root"/>
        <property name="password" value="123456"/>
        <property name="url" value="jdbc:mysql://localhost:3306/spring_data"/>
    </bean>

    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <bean id="studentDao" class="com.sunnywr.dao.StudentDaoSpringJdbcImpl">
        <property name="jdbcTemplate" ref="jdbcTemplate"/>
    </bean>
</beans>
```
使用Spring Jdbc Template的数据库操作实现：
```java
public class StudentDaoSpringJdbcImpl implements StudentDao {

    private JdbcTemplate jdbcTemplate;

    @Override
    public List<Student> queryList() {

        final List<Student> students = new ArrayList<>();
        String sql = "SELECT id, name, age FROM student";

        jdbcTemplate.query(sql, new RowCallbackHandler() {
            @Override
            public void processRow(ResultSet rs) throws SQLException {
                int id = rs.getInt("id");
                String name = rs.getString("name");
                int age = rs.getInt("age");
                students.add(new Student(id, name, age));
            }
        });
        return students;
    }

    @Override
    public void insertOne(Student student) {
        String sql = "INSERT INTO student(name, age) VALUES(?, ?)";
        jdbcTemplate.update(sql, new Object[] {student.getName(), student.getAge()});
    }

    public JdbcTemplate getJdbcTemplate() {
        return jdbcTemplate;
    }

    // Spring set() 方法注入
    public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
}
```
Spring环境配置测试：
```java
public class SpringDatasourceTest {

    private ApplicationContext context = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-context-beans.xml");
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void datasourceTest() {
        DataSource dataSource = (DataSource) context.getBean("dataSource");
        Assert.assertNotNull(dataSource);
    }

    @Test
    public void jdbcTemplateTest() {
        JdbcTemplate jdbcTemplate = (JdbcTemplate) context.getBean("jdbcTemplate");
        Assert.assertNotNull(jdbcTemplate);
    }
}
```
Spring JDBC Template实现的测试：
```java
public class StudentDaoSpringJdbcImplTest {

    private ApplicationContext context = null;
    private StudentDao studentDao = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-context-beans.xml");
        studentDao = (StudentDao) context.getBean("studentDao");
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        studentDao = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void queryListTest() {
        List<Student> students = studentDao.queryList();
        for(Student student : students) {
            System.out.println(student.toString());
        }
    }

    @Test
    public void insertOneTest() {
        Student student = new Student("test-SpringJdbc", 18);
        studentDao.insertOne(student);
    }
}
```

# 传统Dao开发方式的弊端
从上述代码可以看出，不管是传统的JDBC方式还是使用Spring JDBC Temp的方式，Dao层都有着很大的代码量，且有很多重复的代码。而使用Spring JDBC的方式虽然代码量相比而言有所减少，但却依赖复杂的配置，开发与维护成本较高。

再者，假设我们现在需要开发分页等其他功能，在这样的代码基础上进行扩展也十分困难。为了解决这些问题，Spring Data正式诞生了。
