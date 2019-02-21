---
title: Spring事务管理
date: 2016-06-05 09:31:16
tags: [spring,transaction]
categories:
- framework
- Spring
---

> Spring系列文章：
> https://zhum.in/blog/categories/frame/Spring/

> 项目代码：
> https://github.com/njZhuMin/SpringTransaction

- - -
# 事务
## 事务的概念
事务指的是逻辑上的一组操作，这组操作要么全部成功，要么一起失败。

以银行转账为例：
A向B转账1000元，A -1000元，B +1000元，则转账成功；
若A -1000元，B +1000元失败，那么A也不能扣款，转账事务失败。

## 事物的特性：
1. 原子性：事物是一个不可分割的工作单位，事务中的操作要么都发生，要么都不发生。
2. 一致性：指事务前后数据的完整性必须保持一致
3. 隔离性：指多个用户并发访问数据库时，一个用户的事务不能被其他用户的事务所干扰，多个并发事务之间数据要相互隔离。
4. 持久性：一个事务一旦被提交，他对数据库中数据的改变就是永久性的，即使数据库发生故障也不应该对其有任何影响。

<!-- more -->

# Spring事务管理
## 事务管理接口
Spring事务管理高层抽象主要有3个接口。
1. platform TransactionManager：平台事务管理
2. TransactionDefinition：事务定义信息（隔离、传播、超时、只读）
3. TransactionStatus：事务具体运行状态

PlatformTransactionManager 根据 TransactionDefinition 进行事务管理，管理过程中事务存在多种状态，每个状态信息通过 TransactionStatus 表示。

## PlatformTransactionManager
Spring为不同的持久化框架提供了不同的PlatformTransactionManager接口实现。常用的有`org.springframework.jdbc.datasource.DataSourceTransactionManager`（Spring JDBC，MyBatis)和`org.springframework.orm.hibernate5.HibernateTransactionManager
`（Hibernate）。

## TransactionDefinition
`TransactionDefinition`接口管理事务隔离属性。其中定义了一些常量：
- ISOLATION_*：事务的隔离级别
- PROPAGATION_*：事务传播行为
- TIMEOUT_DEFAULT：超时信息

## 事务的隔离级别：
- dirty reads:一个事务读取了另一个事务改写但还未提交的数据，如果这些数据被回滚，则读到的数据是无效的。
- non-repeatable reads：在同一事务中，多次读取同一数据返回的结果有所不同。
- phantom reads：一个事务读取了几行记录后，另一个事务insert一些记录，就发生了虚读（幻读）。在后来的查询中，第一个事务就会发现有些原来没有的记录。

|ISOLATION LEVEL  |DEFINITION |
|-----------------|-----------|
|ISOLATION_DEFAULT | Use the default isolation level of the underlying datastore.|
|ISOLATION_READ_COMMITTED | Indicates that dirty reads are prevented; non-repeatable reads and phantom reads can occur.|
|ISOLATION_READ_UNCOMMITTED | Indicates that dirty reads, non-repeatable reads and phantom reads can occur.|
|ISOLATION_REPEATABLE_READ | Indicates that dirty reads and non-repeatable reads are prevented; phantom reads can occur.|
|ISOLATION_SERIALIZABLE | Indicates that dirty reads, non-repeatable reads and phantom reads are prevented.|

其中`ISOLATION_DEFAULT`级别时，Spring将使用数据库默认的隔离级别。MySQL中的默认级别是`REPEATABLE_READ`，而Oracle中的默认级别是`READ_COMMITTED`。

## 事务的传播行为
项目开发中，通常服务器端分为三层：
```bash
Web层 --> 业务层 --> 持久层
```
例如，在持久层中有两个DAO，而在业务层中有两个Service，其中Service1中需要调用两个DAO的方法，Service2中需要调用另外的方法。
```java
DAO1 { method1(); }
DAO2 { method2(); }
```
```java
Service1 {
	DAO1.method1();
	DAO2.method2();
}
Service2 {
	anotherDAO.anotherMethod();
}
```
在某种复杂业务中，可能会需要同时调用Service1中的部分方法与Service2中的部分方法来完成某个事务。那么事务的传播行为就是为了解决业务层之间的方法相互调用的问题。


事务的传播行为大致分为三类：
1. 支持当前事务，如果当前没有事务，不同的处理：
|PROPAGATION BEHAVIOR | DEFINITION |
|---------------------|------------|
|PROPAGATION_REQUIRED | Support a current transaction; create a new one if none exists.|
|PROPAGATION_SUPPORTS | Support a current transaction; execute non-transactionally if none exists.|
|PROPAGATION_MANDATORY | Support a current transaction; throw an exception if no current transaction exists.|

2. 不和当前方法处于同一个事务，如果当前有事务，不同的处理：
|PROPAGATION BEHAVIOR | DEFINITION |
|---------------------|------------|
|PROPAGATION_REQUIRES_NEW | Create a new transaction, suspending the current transaction if one exists.|
|PROPAGATION_NOT_SUPPORTED | Do not support a current transaction; rather always execute non-transactionally.|
|PROPAGATION_NEVER | Do not support a current transaction; throw an exception if a current transaction exists.|

3. 如果当前事务存在，则设置保存点并嵌套执行事务，如果后一个事务发生异常，则自定义操作：
|PROPAGATION BEHAVIOR | DEFINITION |
|---------------------|------------|
|PROPAGATION_NESTED | Execute within a nested transaction if a current transaction exists, behave like PROPAGATION_REQUIRED else.|

## TransactionStatus
`TransactionStatus`类用于保存事务状态属性，提供方法获取事务状态：
`boolean hasSavepoint()`、`boolean isCompleted()`、`boolean isNewTransaction()`等。

# 编程式事务控制
- 在需要事务控制的类中使用`TransactionTemplate`
- `TransactionTemplate`依赖`DataSourceTransactionManager`
- `DataSourceTransactionManager`依赖`DataSource`构造

```java
public void transfer(final String out, final String in, final Double money) {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus status) {
            accountDao.outMoney(out, money);
            int i = 1 / 0;
            accountDao.inMoney(in, money);
        }
    });
}
```

# 声明式事务管理
## 方式一：基于`TransactionProxyFactoryBean`
```xml
<!-- 声明式1：基于TransactionProxyFactoryBean -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>
<!-- 配置业务层代理 -->
<bean id="accountServiceProxy" class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">
    <property name="target" ref="accountService" />
    <!-- 注入事务管理器 -->
    <property name="transactionManager" ref="transactionManager" />
    <!-- 注入事务属性 -->
    <property name="transactionAttributes">
        <props>
            <!-- props格式：
                PROPAGATION:事务的传播行为
                ISOLATION:事务的隔离级别
                readOnly：只读
                -Exception：发生哪些异常回滚事务
                +Exception：发生哪些异常事务不会滚
            -->
            <prop key="transfer">PROPAGATION_REQUIRED</prop>
        </props>
    </property>
</bean>
```

## 方式二：基于AspectJ的XML方式
通常不使用方式一进行开发，因为这种方式需要对每一个需要代理的事务配置一个事务管理的代理。因此我们使用基于AspectJ的方式。

基于AspectJ时我们使用Advice切面对对象做动态代理：
```xml
<!-- 声明式2：基于AspectJ的XML方式 -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource" />
</bean>

<!-- 配置Advice -->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="transfer" propagation="REQUIRED"/>
    </tx:attributes>
</tx:advice>

<!-- 配置AOP切面 -->
<aop:config>
    <!-- 配置切入点 -->
    <aop:pointcut id="pointcut1" expression="execution(* com.sunnywr.transfer.service.AccountService+.*(..))" />
    <!-- 配置切面 -->
    <aop:advisor advice-ref="txAdvice" pointcut-ref="pointcut1" />
</aop:config>
```

## 方式三：基于注解的事务管理方式
- XML配置：
```xml
<!-- 声明式3：基于注解的事务管理方式 -->
<!-- 配置事务管理器 -->
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
<property name="dataSource" ref="dataSource" />
</bean>
<!-- 允许注解事务 -->
<tx:annotation-driven transaction-manager="transactionManager" />
```

- 为业务层添加注解
```java
@Transactional(propagation = Propagation.REQUIRED)
public class AccountServiceImpl implements AccountService {
    // 注入转账的DAO类
    private AccountDao accountDao;

    public void setAccountDao(AccountDao accountDao) {
        this.accountDao = accountDao;
    }

    public void transfer(String out, String in, Double money) {
        accountDao.outMoney(out, money);
        int i = 1 / 0;
        accountDao.inMoney(in, money);
    }
}
```

# 总结
Spring将事务管理分成`编程式事务管理`和`声明式事务管理`两类。

## 编程式事务管理
手动编写代码进行事务管理（很少使用）

## 声明式事务管理
- 基于TransactionProxyFactoryBean：需要对每一个进行事务管理的类配置TransactionProxyFactoryBean

- 基于AspectJ的XML方式：纯配置方式，不需要修改代码

- 基于注解方式：配置简单，但需要在业务层代码上增加注解
