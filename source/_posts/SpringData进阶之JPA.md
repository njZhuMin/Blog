---
title: SpringData进阶之JPA
date: 2017-6-24 01:00:00
tags: [SpringData]
categories:
- framework
- SpringData
---

# 环境依赖
我们首先配置Spring Data Jpa的依赖：
```xml
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-jpa</artifactId>
    <version>1.11.4.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-entitymanager</artifactId>
    <version>5.2.10.Final</version>
</dependency>
```

<!-- more -->

然后定义Spring相关的配置文件：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:tx="http://www.springframework.org/schema/tx"
       xmlns:jpa="http://www.springframework.org/schema/data/jpa"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/data/jpa http://www.springframework.org/schema/data/jpa/spring-jpa-1.3.xsd
		http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd">

    <!-- 配置数据源 -->
    <bean id="dataSource" class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
        <property name="username" value="root"/>
        <property name="password" value="minmin95420"/>
        <property name="url" value="jdbc:mysql://localhost:3306/spring_data"/>
    </bean>

    <!-- 配置EntityManager -->
    <bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
        <property name="dataSource" ref="dataSource"/>
        <property name="jpaVendorAdapter">
            <bean class="org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter"/>
        </property>
        <property name="packagesToScan" value="com.sunnywr"/>

        <property name="jpaProperties">
            <props>
                <prop key="hibernate.ejb.naming_strategy">org.hibernate.cfg.ImprovedNamingStrategy</prop>
                <prop key="hibernate.dialect">org.hibernate.dialect.MySQL5InnoDBDialect</prop>
                <prop key="hibernate.show_sql">true</prop>
                <prop key="hibernate.format_sql">true</prop>
                <prop key="hibernate.hbm2ddl.auto">update</prop>
            </props>
        </property>
    </bean>

    <!-- 配置事务管理器 -->
    <bean id="transactionManager" class="org.springframework.orm.jpa.JpaTransactionManager">
        <property name="entityManagerFactory" ref="entityManagerFactory"/>
    </bean>

    <!-- 配置支持注解的事务-->
    <tx:annotation-driven transaction-manager="transactionManager"/>

    <!-- 配置spring data-->
    <jpa:repositories base-package="com.sunnywr" entity-manager-factory-ref="entityManagerFactory"/>
    <context:component-scan base-package="com.sunnywr"/>
</beans>
```
接下来我们定义一个实体类，然后写一个测试类来测试实体类对应数据表的生成：
```java
@Entity(name = "employee")
public class Employee {

    @GeneratedValue
    @Id
    private Integer id;

    @Column(length = 20, nullable = false)
    private String name;

    @Column(nullable = false)
    private Integer age;
    // ... getters and setters and toString()
}

public class SpringDataJpaTest {

    private ApplicationContext context = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-data-jpa.xml");
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void EntityManagerFactoryTest() {

    }
}
```
可以看到，虽然这里的测试函数是空的，但随着Spring的加载，在数据库中自动建立了Employee实体类对应的数据表employee。

接下来我们只需要继承Spring Data Jpa的Repository接口，并且声明相关的方法，即可自动完成数据库调用操作：
```java
public interface EmployeeRepository extends Repository<Employee, Integer> {

    public Employee findByName(String name);
}

public class EmployeeRepositoryTest {

    private ApplicationContext context = null;
    private EmployeeRepository employeeRepository = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-data-jpa.xml");
        employeeRepository = context.getBean(EmployeeRepository.class);
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        employeeRepository = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void findByNameTest() {
        Employee employee = employeeRepository.findByName("Tom");
        System.out.println(employee.toString());
        Assert.assertTrue(employee.getId() == 1);
    }
}
```
经过测试，我们仅通过继承Spring Data Jpa提供的Repository接口，声明了一个函数，就成功的从数据库中按姓名查询出了我们需要的条目。

# Repository接口
## 概述
我们先来看看Spring Data中的Repository接口是什么：
- Repository接口是Spring Data的核心接口，它并不提供任何方法
- 原型 public interface Repository<T, ID extends Serializable> {}
- 不一定必须继承此接口，也可以使用`@RepositoryDefinition`注解实现

可以看到，Repository是一个空接口（标记接口），本身不包含任何方法。如果我们自定义的接口继承了Repository，那么该接口会被Spring自动接管（SimpleJpaRepository）：
```bash
org.springframework.data.jpa.repository.support.SimpleJpaRepository@2c6ee758
```
如果我们的接口未继承自Repository，那么运行时会报错：
```bash
org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'com.imooc.repository.EmployeeRepository' available
```
即这个接口并没有被Spring接管，因此无法从Spring Bean中获取。我们也可以使用注解来实现Spring的托管：
```java
@RepositoryDefinition(domainClass = Employee.class, idClass = Integer.class)
public interface EmployeeRepository {
    public Employee findByName(String name);
}
```

## Repository的子接口
常用的Repository接口的实现有：
- CrudRepository：实现了CRUD相关方法
- PagingAndSortingRepository：继承CrudRepository，实现了分页排序的相关方法
- JPARepository：继承PagingAndSortingRepository，实现JPA相关规范

## 查询方法定义
Spring Data中的查询方法命名必须遵循一定的规则：
```java
public interface EmployeeRepository extends Repository<Employee, Integer> {
    // WHERE name LIKE ?% AND age < ?
    public List<Employee> findByNameStartingWithAndAgeLessThan(String name, Integer age);

    // WHERE name LIKE %? AND age < ?
    public List<Employee> findByNameEndingWithAndAgeLessThan(String name, Integer age);

    // WHERE name IN (?,?,...) OR age < ?
    public List<Employee> findByNameInOrAgeLessThan(List<String> names, Integer age);

    // WHERE name IN (?,?,...) AND age < ?
    public List<Employee> findByNameInAndAgeLessThan(List<String> names, Integer age);
}
```
测试类：
```java
public class EmployeeRepositoryTest {
    @Test
    public void findByNameStartingWithAndAgeLessThanTest() {
        List<Employee> employees = employeeRepository.findByNameStartingWithAndAgeLessThan("test", 22);

        for(Employee employee : employees)
            System.out.println(employee);
    }

    @Test
    public void findByNameEndingWithAndAgeLessThanTest() {
        List<Employee> employees = employeeRepository.findByNameEndingWithAndAgeLessThan("6", 25);

        for(Employee employee : employees)
            System.out.println(employee);
    }

    @Test
    public void findByNameInOrAgeLessThanTest() {
        List<String> names = new ArrayList<>();
        names.add("test1");
        names.add("test2");
        names.add("test3");
        List<Employee> employees = employeeRepository.findByNameInOrAgeLessThan(names, 22);

        for(Employee employee : employees)
            System.out.println(employee);
    }

    @Test
    public void findByNameInAndAgeLessThanTest() {
        List<String> names = new ArrayList<>();
        names.add("test1");
        names.add("test2");
        names.add("test3");
        List<Employee> employees = employeeRepository.findByNameInAndAgeLessThan(names, 22);

        for(Employee employee : employees)
            System.out.println(employee);
    }
}
```
这样的实现方法虽然简单，但是也有一些弊端：
1. 方法名会比较长，约定规则大于配置
2. 对于一些复杂的查询很难实现
对此Spring Data提供了`@Query`注解来解决这一问题。

## @Query注解
`@Query`注解的使用很简单，我们只要：
- 在Repository方法中使用，就可以不需要遵循查询方法命名规则
- 只需要将`@Query`定义在Repository中的方法之上
- 命名参数及索引参数的使用

使用HQL命名参数写法：
```java
public interface EmployeeRepository extends Repository<Employee, Integer> {
    @Query("SELECT obj FROM employee obj WHERE id=(SELECT MAX(id) FROM employee t1)")
    public Employee getEmployeeByMaxId();

    @Query("SELECT obj FROM employee obj WHERE obj.name=?1 AND obj.age=?2")
    public Employee queryParams1(String name, Integer age);

    @Query("SELECT obj FROM employee obj WHERE obj.name=:name AND obj.age=:age")
    public List<Employee> queryParams2(@Param("name")  String name, @Param("age") Integer age);

    @Query("SELECT obj FROM employee obj WHERE obj.name LIKE %?1%")
    public List<Employee> queryLike1(String name);

    @Query("SELECT obj FROM employee obj WHERE obj.name LIKE %:name%")
    public List<Employee> queryLike2(@Param("name") String name);
}
```

使用SQL原生查询写法：
```java
public interface EmployeeRepository extends Repository<Employee, Integer> {
    @Query(nativeQuery = true, value = "SELECT COUNT(1) FROM employee")
    public long queryCount();
}
```

## 更新和删除事务
当我们操作更新或删除时，我们需要结合`@Modifying`、`@Query`和`@Transactional`注解来实现。
```java
public interface EmployeeRepository extends Repository<Employee, Integer> {
    @Modifying
    @Query("UPDATE employee obj SET obj.age=:age WHERE obj.id=:id")
    public void update(@Param("id") Integer id, @Param("age") Integer age);
}

@Service
public class EmployeeService {

    @Autowired
    private EmployeeRepository employeeRepository;

    @Transactional
    public void update(Integer id, Integer age) {
        employeeRepository.update(1, 40);
    }
}

public class EmployeeServiceTest {

    private ApplicationContext context = null;
    private EmployeeService employeeService = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-data-jpa.xml");
        employeeService = context.getBean(EmployeeService.class);
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        employeeService = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void updateTest() throws Exception {
        employeeService.update(1,55);
    }
}
```
这里需要注意的是：
1. 更新与删除需要`@Modifying`注解
2. 必须配合`@Transactional`注解配置事务
3. 事务逻辑一般是在Service层（反复调用多个Dao）

# JPA高级接口
## CrudRepository
`CrudRepository`接口提供了常见的CRUD的操作方法，具体使用如下：
```java
public interface EmployeeCrudRepository
        extends CrudRepository<Employee, Integer> {
}

@Service
public class EmployeeService {

    @Autowired
    private EmployeeRepository employeeRepository;

    @Autowired
    private EmployeeCrudRepository employeeCrudRepository;

    @Transactional
    public void saveMulti(List<Employee> employees) {
        employeeCrudRepository.save(employees);
    }
}

public class EmployeeCrudRepositoryTest {

    private ApplicationContext context = null;
    private EmployeeCrudRepository employeeCrudRepository = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-data-jpa.xml");
        employeeCrudRepository = context.getBean(EmployeeCrudRepository.class);
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        employeeCrudRepository = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void saveMultiTest() {
        List<Employee> employees = new ArrayList<Employee>();
        Employee employee = null;
        for(int i = 20; i < 40; i++) {
            employee = new Employee("test-"+i, 50-i);
            employees.add(employee);
        }

        employeeCrudRepository.save(employees);
    }
}
```

## 分页排序接口
JPA中提供了`PagingAndSortingRepository`接口，实现了排序和分页功能。方法声明为`findAll(Sort sort)'和'findAll(Pageable pageable)`。
```java
public interface EmployeePagingAndSortingRepository
        extends PagingAndSortingRepository<Employee, Integer> {
}

public class EmployeePagingAndSortingRepositoryTest {

    private ApplicationContext context = null;
    private EmployeePagingAndSortingRepository employeePagingAndSortingRepository = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-data-jpa.xml");
        employeePagingAndSortingRepository = context.getBean(EmployeePagingAndSortingRepository.class);
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        employeePagingAndSortingRepository = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void pageTest() {

        Pageable pageable = new PageRequest(0, 8);
        Page<Employee> page = employeePagingAndSortingRepository.findAll(pageable);

        System.out.println("Total page: " + page.getTotalPages());
        System.out.println("Total elements: " + page.getTotalElements());
        System.out.println("Now page: " + page.getNumber());
        System.out.println("Now contents: " + page.getContent());
        System.out.println("Now elements on page: " + page.getNumberOfElements());
    }

    @Test
    public void pageAndSortTest() {

        Sort.Order order = new Sort.Order(Sort.Direction.DESC, "id");
        Sort sort = new Sort(order);

        Pageable pageable = new PageRequest(0, 8, sort);
        Page<Employee> page = employeePagingAndSortingRepository.findAll(pageable);

        System.out.println("Total page: " + page.getTotalPages());
        System.out.println("Total elements: " + page.getTotalElements());
        System.out.println("Now page: " + page.getNumber());
        System.out.println("Now contents: " + page.getContent());
        System.out.println("Now elements on page: " + page.getNumberOfElements());
    }
}
```

## JpaRepository接口
```java
public interface EmployeeJpaRepository
        extends JpaRepository<Employee, Integer> {
}

public class EmployeeJpaRepositoryTest {

    private ApplicationContext context = null;
    private EmployeeJpaRepository employeeJpaRepository = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-data-jpa.xml");
        employeeJpaRepository = context.getBean(EmployeeJpaRepository.class);
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        employeeJpaRepository = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void findTest() {

        Employee employee1 = employeeJpaRepository.findOne(10);
        System.out.println(employee1);

        Employee employee2 = employeeJpaRepository.findOne(54);
        System.out.println(employee2);
    }
}
```

## JpaSpecificationExcutor接口
`Specification`接口提供了JPA Criteria的实现。
```java
public interface EmployeeJpaSpecificationExecutorRepository
    extends JpaRepository<Employee, Integer>, JpaSpecificationExecutor<Employee> {
}

public class EmployeeJpaSpecificationExecutorRepositoryTest {

    private ApplicationContext context = null;
    private EmployeeJpaSpecificationExecutorRepository employeeJpaSpecificationExecutorRepository = null;

    @Before
    public void initContext() {
        context = new ClassPathXmlApplicationContext("spring-data-jpa.xml");
        employeeJpaSpecificationExecutorRepository = context.getBean(EmployeeJpaSpecificationExecutorRepository.class);
        System.out.println("Context initialized...");
    }

    @After
    public void destroyContext() {
        context = null;
        employeeJpaSpecificationExecutorRepository = null;
        System.out.println("Context destroyed...");
    }

    @Test
    public void findTest() {

        // 1. 分页
        // 2. 排序
        // 3. 条件：age > 20
        Sort.Order order = new Sort.Order(Sort.Direction.DESC, "id");
        Sort sort = new Sort(order);

        Pageable pageable = new PageRequest(0, 8, sort);

        /**
         * root: 查询类型：Employee
         * query： 添加查询条件
         * cb： 构建predicate对象
         */
        Specification<Employee> specification = new Specification<Employee>() {
            @Override
            public Predicate toPredicate(Root<Employee> root, CriteriaQuery<?> query, CriteriaBuilder cb) {
                // 相当于 root(employee(age))
                Path path = root.get("age");
                return cb.gt(path, 20);
            }
        };

        Page<Employee> page = employeeJpaSpecificationExecutorRepository.findAll(specification, pageable);

        System.out.println("Total page: " + page.getTotalPages());
        System.out.println("Total elements: " + page.getTotalElements());
        System.out.println("Now page: " + page.getNumber());
        System.out.println("Now contents: " + page.getContent());
        System.out.println("Now elements on page: " + page.getNumberOfElements());
    }
}
```
