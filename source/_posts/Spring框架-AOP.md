---
title: Spring框架-AOP
date: 2016-06-03 09:31:16
tags: [spring,AOP]
categories:
- framework
- Spring
---

> Spring系列文章：
> https://zhum.in/blog/categories/frame/Spring/

> 项目代码：
> https://github.com/njZhuMin/SpringDemo

- - -
# AOP
## 面向切面编程
AOP(Aspect Oriented programming)：面向切面编程，通过预编译方式和运行期动态代理实现程序功能的统一维护的一种技术。
主要的功能是：日志记录，性能统计，安全控制，事务处理，异常处理等等。

## AOP实现方式
1. 预编译 - AspectJ
2. 运行期动态代理（JDK动态代理、CGLib动态代理）
SpringAOP、JbossAOP

> 有接口的SpringAOP实现使用JDK动态代理作为AOP代理，这使得任何接口（或接口集）都可以被代理；
> 如果一个业务对象并没有实现一个接口，Spring AOP中也可以使用CGLIB代理。

<!-- more -->

## AOP通知（Advice）类型
1. 前置通知（before advice），在JointPoint前执行

2. 返回后通知（after returning advice），在JointPoint后执行

3. 抛出异常后通知（after throwing advice），在JointPoint抛出异常时执行

4. 后通知：（after[finally] advice），在JointPoint调用完毕后执行

5. 环绕通知：（around advice），在JointPoint前后执行

## Spring框架中AOP
- 用途
1. 提供了声明式的企业服务，特别是EJB的替代服务的声明
2. 允许用户定制自己的方面，以完成OOP与AOP的互补使用

- 实现
1. 纯Java实现，无需特殊的编译过程，不需要控制类加载器层次
2. 目前只支持方法执行连接点（通知Spring Bean的方法执行）
3. 不是为了提供最完整的AOP实现，而是侧重于提供一种AOP实现和Spring IoC容器之间的整合，用于帮助解决企业应用中的常见问题
4. Spring AOP不会与AspectJ竞争，从而提供综合全面的AOP解决方案

## Schema-based AOP
Spring所以的切面和通知器都必须放在一个`<aop:config>`内（可以配置包含多个`<aop:config>`元素，每一个`<aop:config>`可以包含`pointcut`，`advisor`和`aspect`元素（必须按照这个顺序进行声明）。

- 环绕通知（around advice）的方法第一个参数必须为`ProceedingJoinPoint`类型。

```java
public class AspectBiz {
    public void biz() {
        System.out.println("AspectBiz.biz()");
//        throw new RuntimeException();
    }
}
```
```java
public class MyAspect {
    public void before() {
        System.out.println("MyAspect before...");
    }

    public void afterReturning() {
        System.out.println("MyAspect afterReturning...");
    }

    public void afterThrowing() {
        System.out.println("MyAspect afterThrowing...");
    }

    public void after() {
        System.out.println("MyAspect after...");
    }

    public Object around(ProceedingJoinPoint pjp) {
        Object object = null;
        try {
            System.out.println("MyAspect around 1...");
            object = pjp.proceed();
            System.out.println("MyAspect around 2...");
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }
        return object;
    }
}
```
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop.xsd">

    <bean id="myAspect" class="com.sunnywr.aop.schema.advice.MyAspect"></bean>
    <bean id="aspectBiz" class="com.sunnywr.aop.schema.advice.biz.AspectBiz"></bean>
    <aop:config>
        <aop:aspect id="myAspectAOP" ref="myAspect">
            <aop:pointcut id="myPointcut" expression=
                    "execution(* com.sunnywr.aop.schema.advice.biz.*Biz.*(..))" />
            <aop:before method="before" pointcut-ref="myPointcut" />
            <aop:after-returning method="afterReturning" pointcut-ref="myPointcut" />
            <aop:after-throwing method="afterThrowing" pointcut-ref="myPointcut" />
            <aop:after method="after" pointcut-ref="myPointcut" />
            <aop:around method="around" pointcut-ref="myPointcut" />
        </aop:aspect>
    </aop:config>
</beans>
```

## Advice Parameters
在Advice Parameters中指定具体某个方法（带参数）的配置方式，参数可以被对应的advice方法使用。以aroud为例：
```xml
<aop:around method="aroundInit" pointcut="execution(* com.sunnywr.aop.schema.advice.biz.AspectBiz.init(String, int))	and args(bizName, times)"/>
```
- AspectBiz.java

```java
public void init(String bizName, int times) {
    System.out.println("AspectBiz init: " + bizName + ", " + times);
}
```
- MyAspect.java

```java
public Object aroundInit(ProceedingJoinPoint pjp, String bizName, int times) {
    System.out.println(bizName + ", " + times);
    Object object = null;
    try {
        System.out.println("MyAspect aroundInit 1...");
        object = pjp.proceed();
        System.out.println("MyAspect aroundInit 2...");
    } catch (Throwable throwable) {
        throwable.printStackTrace();
    }
    return object;
}
```
- Test方法

```java
@Test
public void testInit() {
    AspectBiz biz = super.getBean("aspectBiz");
    biz.init("myService", 3);
}
```

## Introductions
- 简介通知（Introductions）允许一个切面声明一个实现指定接口的通知对象，并且提供了一个接口实现类来代表这些对象。

- Introductions由`<aop:aspect>`中的`<aop:declare-parents>`元素声明该元素用于声明所匹配的类型拥有一个新的parent。

- 对于`<declare-parents>`的作用
```xml
<aop:declare-parents types-matching="com.sunnywr.aop.schema.advice.biz.*(+)"
    implement-interface="com.sunnywr.aop.schema.advice.Fit"
    default-impl="com.sunnywr.aop.schema.advice.FitImpl" />
```
在这里，declare-parents 为types-matching中的类（用proxy表示）指定了一个父类，然后又在指定了此父类为接口interface，并指出此父类的一个默认实现类impl。

 这个运用的属于代理模式中的静态代理，本质上就是通过proxy代理了impl，实现并可以加强imple中的功能。假如说impl中只有一个方法`method()`，那么proxy就可以代理`method()`并为其添加附加功能、设定访问权限等等。

> schema-defined aspects只支持singleton mode

## Advisors
- advisor就像一个小的自包含的方面，只有一个advice

- 切面自身通过一个bean表示，并且必须实现某个advice接口，同时，advisor也可以很好的利用AspectJ的切入表达式

- Spring通过配置文件中`<aop:advisor>`元素支持advisor实际使用中，大多数情况下它会和`transactional advice`配合使用

- 为了定义一个advisor的优先级以便让advice可以有序，可以使用order属性来定义advisor的顺序

# Spring AOP API
## Pointcut
实现之一：NameMatchMethodPointcut（根据方法名字匹配）
成员变量：mappedNames，匹配的方法名集合：
```xml
<bean id="pointcutBean" class="org.springframework.aop.support.NameMatchMethodPointcut">
    <property name="mappedNames">
        <list>
            <value>sa*</value>
        </list>
    </property>
</bean>
```

## Before Advice
只在进入方法之前被调用，不需要`MethodInvocation`对象。前置通知可以在连接点之前插入自定义行为，但不能改变返回值。

Before Advice需要实现`MethodBeforeAdvice`接口：
```java
public class MyBeforeAdvice implements MethodBeforeAdvice {
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("MyBeforeAdvice: " + method.getName()
                + ", target: " + target.getClass().getName());
    }
}
```

## Throws Advice
- 如果连接点抛出异常，throws advice在连接点返回后被调用

- 如果throws-advice的方法抛出异常，将覆盖原有异常

- 接口`org.springframework.aop.ThrowsAdvice`不包含任何方法，仅仅是一个声明，但需要实现类实现类似以下方法：
`void afterThrowing([Method, args, target], ThrowableSubclass);`

```java
public class MyThrowsAdvice implements ThrowsAdvice {
    public void afterThrowing(Exception ex) throws Throwable {
        System.out.println("MyThrowsAdvice afterThrowing 1");
    }

    public void afterThrowing(Method method, Object[] args, Object target, Exception ex)
            throws Throwable {
        System.out.println("MyThrowsAdvice afterThrowing 2: "
            + method.getName() + " " + target.getClass().getName());
    }
}
```

## After Returning Advice
- 后置通知必须实现`org.springframework.aop.AfterReturningAdvice`接口

- 可以访问返回值（但不能修改）、被调用的方法、方法的参数和目标

- 如果抛出异常，将会抛出拦截器链，替代返回值

```java
public class MyAfterReturningAdvice implements AfterReturningAdvice {
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target) throws Throwable {
        System.out.println("MyAfterReturningAdvice afterReturning: "
                + method.getName() + " " + target.getClass().getName() + " " + returnValue);
    }
}
```

## Interception Around Advice
Spring的切入点模型使得切入点可以独立与Advice重用，以针对不同的Advice可以使用相同的切入点。

```java
public class MyMethodInterceptor implements MethodInterceptor {
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("MyMethodInterceptor 1: " + invocation.getMethod().getName()
            + " " + invocation.getStaticPart().getClass().getName());
        Object obj = invocation.proceed();
        System.out.println("MyMethodInterceptor 2: " + obj);
        return obj;
    }
}
```

## Introduction Advice
- Spring把引入通知作为一种特殊的拦截通知

- 需要`Introduction Advisor`和`Introduction Interceptor`

- 仅适用于类，不能和任何切入点一起使用

> - 如果调用`lock`方法，希望所有`setter`方法抛出`LockedException`异常，使对象不可改变；

> - 需要一个完成繁重任务的`IntroductionInterceptor`，这种情况下可以使用`org.springframework.aop.support.DelegatingIntroductionInterceptor;`

```java
public interface Lockable {
    void lock();
    void unlock();
    boolean isLocked();
}

public class LockMixin extends DelegatingIntroductionInterceptor implements Lockable {
    private static final long serialVersionUID = 6943163819932660450L;
    private boolean locked;
    public void lock() {
        this.locked = true;
    }
    public void unlock() {
        this.locked = false;
    }

    public boolean isLocked() {
        return this.locked;
    }
    public Object invoke(MethodInvocation invocation) throws Throwable {
        if(isLocked() && invocation.getMethod().getName().indexOf("set") == 0) {
            throw new RuntimeException();
        }
        return super.invoke(invocation);
    }
}

public class LockMixinAdvisor extends DefaultIntroductionAdvisor {
    private static final long serialVersionUID = -171332350782163120L;
    public LockMixinAdvisor() {
        super(new LockMixin(), Lockable.class);
    }
}
```

## Advisor API
- Advisor是仅包含一个切入点表达式关联的单个通知方面

- 除了introductions，advisor可以用于任何通知

- `org.springframework.aop.support.DefaultPointcutAdvisor`是最常用的advisor类，它可以与`MethodInterceptor`，`BeforeAdvice`或`ThrowsAdvice`一起使用

- 它可以混合在Spring同一个AOP代理的advisor和advice

## ProxyFactoryBean
创建String AOP代理的基本方法是使用`org.springframework.aop.framework.ProxyFactoryBean`，这可以完全控制切入点和通知（advice）以及他们的顺序。

例如，当我们向`ProxyFactoryBean`申请一个变量实例foo时，并不会获得一个`ProxyFactoryBean`实例，而是获得`ProxyFactoryBean`实现里的`getObject()`方法创建的对象。`getObject()`方法将创建一个AOP代理包装一个目标对象。

- 使用``ProxyFactoryBean`或者其他IoC相关类来创建AOP代理的最重要的好处是通知和切入点也可以由IoC来管理

- 被代理类没有实现任何接口，使用CGLIB代理，否则使用JDK代理

- 通过设置`proxyTargetClass`为true，可以强制使用CGLIB

- 如果目标类实现了一个或者多个接口，那么创建代理的类型将依赖于`ProxyFactoryBean`的配置

- 如果`ProxyFactoryBean`的`proxyInterfaces`属性被设置位一个或多个全限定接口名，则会创建基于JDK代理。如果`ProxyFactoryBean`的`proxyInterfaces`属性没有被设置，但是目标类实现了一个或者多个接口，那么`ProxyFactoryBean`将自动检测这个目标类已经实现了至少一个接口，从而创建一个基于JDK的代理。

## 代理目标与内部匿名Bean
- 代理目标

```xml
<bean id="bizLogicImplTarget" class="com.sunnywr.aop.api.BizLogicImpl"></bean>

<bean id="bizLogicImpl"  parent="baseProxyBean">
    <property name="target">
		<ref bean="bizLogicImplTarget" />
    </property>
</bean>
```

- 内部匿名Bean

```xml
<bean id="bizLogicImpl"  parent="baseProxyBean">
    <property name="target">
		<ref bean="bizLogicImplTarget" />
    </property>
</bean>
```

- 两种方式的区别

在实现上两种方式没有区别，但是使用代理目标方式会暴露对象的bean类，可以在程序中直接获得该类的实例，可能会引起安全问题。因此推荐使用内部匿名Bean的方式来配置代理目标。

## Proxying classes
在一些情况下，也可以强制使用CGLIB方式代理，即使目标类实现了接口。

CGLIB代理的工作原理是在运行时生成目标类的子类，Spring配置这个生成的子类委托方法调用到原来的目标。子类用来实现`Decorator`模式织入通知。

CGLIB的代理对用户是透明的，但是：

- final方法不能被通知，因为final方法不能被覆盖

- 不用手动引入CGLIB的依赖，在Spring 3.2后CGLIB被重新包装并包含在`Spring-core`包中

## global advisors
在配置文件中使用通配符，匹配所有拦截器并加入通知链：
```xml
<bean id="bizLogicImpl"  parent="baseProxyBean">
    <property name="target">
        <bean class="com.sunnywr.aop.api.BizLogicImpl"></bean>
        <!--<ref bean="bizLogicImplTarget" />-->
    </property>
    <property name="proxyInterfaces">
        <value>com.sunnywr.aop.api.BizLogic</value>
    </property>
    <property name="interceptorNames">
        <list>
			<value>my*</value>
        </list>
    </property>
</bean>
```

> 通配符只会将拦截器加入通知链，也即实现了`MethodInterceptor`接口的类。

## 简化的proxy定义
使用父子bean定义，以及内部bean定义，可能会带来更简洁的代理定义（抽象属性标记父bean，定义为抽象的，这样它就不能被实例化）
```xml
<bean id="baseProxyBean" class="org.springframework.aop.framework.ProxyFactoryBean"
    lazy-init="true" abstract="true">
</bean>
```

## 使用ProxyFactory
- 使用Spring AOP而不必依赖于Spring IoC
```java
ProxyFactory factory = new ProxyFactory(myBusinessInterfaceImpl);
factory.addAdvice(myMethodInterceptor);
factory.addAdvisor(myAdvisor);
MyBusinessInterface tb = (MyBusinessInterface) factory.getProxy();
```

- 大多数情况下最佳实践是用IoC容器创建AOP代理

- 虽然可以使用硬编码方式实现，但是Spring推荐使用注解方式实现

## 使用auto-proxy
- Spring也允许使用自动代理的bean定义，它可以自动代理选定的bean。这是建立在Spring的`bean post processor`功能基础上的（在加载bean的时候就可以修改）。

- BeanNameAutoProxyCreator
```xml
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator">
	<property name="beanNames" value="jdk*,onlyJdk" />
	<property name="interceptorNames">
		<list>
			<value>myInterceptor</value>
		</list>
	</property>
</bean>
```

## DefaultAdvisorAutoProxyCreator
当前IoC容器中自动应用，不用显示声明应用advisor的bean定义。

```xml
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator"/>

<bean class="org.springframework.transaction.interceptor.TransactionAttributeSourceAdvisor">
	<property name="transactionInterceptor" ref="transactionInterceptor"/>
</bean>

<bean id="customAdvisor" class="com.mycompany.MyAdvisor"/>

<bean id="businessObject1" class="com.mycompany.BusinessObject1">
<!-- Properties omitted -->
</bean>

<bean id="businessObject2" class="com.mycompany.BusinessObject2"/>
```

# Spring对AspectJ的支持
## AspectJ
- `@AspectJ`的风格类似纯Java注解的普通Java类

- Spring可以使用AspectJ来做切入点解析

- AOP的运行时仍旧时纯Spring AOp，对AspectJ的编译器或者织入无依赖性

## Spring中配置@AspectJ
- 对@AspectJ的支持可以使用xml或Java注解

```java
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```
```xml
<aop:aspectj-autoproxy />
```

- 确保引入AspectJ的`aspectjweaver.jar`依赖

## @Aspect注解
@AspectJ切面使用`@Aspect`注解配置，拥有`@Aspect`的任何bean将被Spring自动识别并应用。

用`@Aspect`注解的类可以有方法和字段，它们也可能包括切入点（pointcut）、通知（advice）和引入（introduction）声明。

> @Aspect注解是不能够通过类路径自动扫描得到的，所以需要配合使用@Component注释或者在bean中配置bean。一个类中的@Aspect注解标识他为一个切面，并且将自己从自动代理中排除。

## pointcut
- 一个切入点通过一个普通的方法定义来提供，并且切入点表达式使用`@Pointcut`注解，方法返回类型必须为void

- 定义一个名为`anyOldTransfer`，这个切入点将匹配任何名为`transfer`的方法的执行：
```java
@Pointcut("execution(* transfer(..))")
private void anyOldTransfer() {
}
```

## 组合pointcut
- 切入点表达式可以通过`&&`、`||`和`!`进行组合，也可以通过名字引用切入点表达式

- 通过组合，可以建立更加复杂的切入点表达式

```java
@Pointcut("execution(public * (..))")
private void anyPublicOperation() {

}

@Pointcut("within(com.example.someapp.trading..)")
private void inTrading() {

}

@Pointcut("anyPublicOperation() && inTrading()")
private void tradingOperation() {

}
```

## 定义良好的pointcuts
- AspectJ是编译期的AOP

- 检查代码并匹配连接点与切入点的代价是昂贵的

- 一个好的切入点应该包括以下几点：
 - 选择特定类型的连接点，如execution、get、set、call、handler
 - 确定连接点范围，如within、withincode
 - 匹配上下文信息，如this、target、@annotation

## AfterReturning Advice
有时需要通知体内得到返回的实际值，可以使用`@AfterReturning`注解绑定具体的返回值。
```java
@Aspect
public class AfterReturningExample {
	@AfterReturning("com.example.myapp.SystemArchitecture.dataAccessOperation()", returning="retVal")
	public void doAccessCheck(Object retVal) {

	}
}
```

## AfterThrowing Advice
有时需要通知体内得到返回的异常，可以使用`@AfterReturning`注解绑定抛出的异常。
```java
@Aspect
public class AfterThrowingExample {
	@AfterReturning("com.example.myapp.SystemArchitecture.dataAccessOperation()", returning="ex")
	public void doRecoveryActions(DataAccessException ex) {

	}
}
```

## After(finally) Advice
最终通知必须准备处理正常和异常两种返回情况，通常用于释放资源。
```java
@Aspect
public class AfterFinallyExample {
	@After("com.example.myapp.SystemArchitecture.dataAccessOperation()")
	public void doReleaseLock() {

	}
}
```

## Around Advice
- 环绕通知使用`@Around`注解来声明，通知方法的第一个参数必须是`ProceedingJoinPoint`类型

- 在通知内部调用`ProceedingJoinPoint`的`proceed()`方法会导致执行真正的方法时传入一个Object[]对象，数组中的值将被作为参数传递给方法。

```java
@Aspect
public class AroundExample {
	@Around("com.example.myapp.SystemArchitecture.businessService()")
	public void doBasicProfilingk(ProceedingJoinPoint pjp) throws Throwable {
		//start stopwatch
		Object retVal = pjp.proceed();
		//stop stopwatch
		return retVal;
	}
}
```

## Advice传递参数
1. 直接传递具体的参数

2. 通过注解的方式传递参数：@Before、@AfterReturning、@AfterThrowing、@After、@Round

3. Spring AOP可以处理泛型类的声明和使用方法的参数

## Advice参数名称
- 通知和切入点注解的`argNames`属性可以指定所注解方法的参数名

- 如果第一个参数是JoinPoint，ProceedingJoinPoint，JoinPoint.StaticPart，那么就可以忽视它

## Introductions
- 允许一个切面声明一个通知对象实现指定接口，并且提供了一个接口实现类来代表这些对象

- introduction使用`@DeclareParents`进行注解，这个注解用来定义匹配的类型拥有一个新的parent

```java
@Aspect
public class UsageTracking {
	@DeclareParents(value="com.example.myapp.service.*+", defaultImpl=DefaultUsageTracked.class)
	public static UsageTracked mixin;

	@Before("com.example.myapp.SystemArchitecture.businessService() && this(usageTracked)")
	public void recordUsage(UsageTracked usageTracked) {
		usageTracked.incrementUseCount();
	}
}
```

# 切面实例化模型
- `perthis`切面通过指定`@Aspect`注解perthis子句实现

- 每个独立的service对象执行时都会创建一个切面实例

- service对象的每个方法在第一次执行的时候创建切面实例，切面在service对象失效的同时失效。

```java
@Aspect("perthis(com.example.myapp.SystemArchitecture.businessService())")
public class MyAspect {
	private int someState;

	@Before(com.example.myapp.SystemArchitecture.businessService())
	public void recordServiceUsage() {

	}
}
```
