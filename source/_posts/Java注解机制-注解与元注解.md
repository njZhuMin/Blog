---
title: Java注解机制-注解与元注解
date: 2016-05-07 19:26:28
tags: [Java,annotation]
categories:
- Java
- annotation
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

- - -

# 注解是什么
注解(Annotation)是Java 1.5中引入的特性，它提供了一种源程序的元素关联任何信息和任何元数据的途径和方法。
Java中内置的注解包括`@Override`、`@Deprecated`、`@Suppvisewarnings`。
## @Override：继承的方法

```java
public interface Person {
	public String name();
	public int age();
	public void sing();
}

public class Child implements Person {
	@Override
	public String name() {
		return null;
	}

	@Override
	public int age() {
		return 0;
	}

	@Override
	public void sing() {
		return;
	}
}
```

<!-- more -->

## @Deprecated：过时的方法
```java
public interface Person {
	public String name();
	public int age();
	@Deprecated
	public void sing();
}

public class Test {
	public static void main(String[] args) {
		Person person = new Child();
		//function sing() is deprecated
		p.sing();
	}
}
```

## @SuppressWarnings：忽略Deprecated等警告
```java
public interface Person {
	public String name();
	public int age();
	@Deprecated
	public void sing();
}

public class Test {
	@SuppressWarnings
	public static void main(String[] args) {
		Person person = new Child();
		//function sing() is deprecated, yet warnings ommitted
		p.sing();
	}
}
```

## 常见的第三方注解
- Spring
 - @Autowired
 - @Service
 - @Repository

- Mybatis
 - @InsertProvider
 - @UpdateProvider
 - @Optionsj

# 注解的分类
## 按运行机制
- 源码注解：注解只在源码中存在，编译成`.class`文件就不存在了

- 编译时注解：注解在源码和.class文件中都存在（如JDK内置系统注解）

- 运行时注解：在运行阶段还起作用，甚至会影响运行逻辑的注解（如Spring中@Autowried）

## 按来源分
- 来自JDK的注解

- 来自第三方的注解

- 自定义注解、

# 元注解
元注解的作用就是负责注解其他注解。Java 1.5定义了4个标准的`meta-annotation`类型，它们被用来提供对其它注解类型作说明：
> 1. @Target

> 2. @Retention

> 3. @Documented

> 4. @Inherited

这些类型和它们所支持的类在java.lang.annotation包中可以找到。

## @Target
@Target说明了Annotation所修饰的对象范围。Annotation可被用于 packages、types（类、接口、枚举、Annotation类型）、类型成员（方法、构造方法、成员变量、枚举值）、方法参数和本地变量（如循环变量、catch参数）。在Annotation类型的声明中使用了target可更加明晰其修饰的目标。

@Target的取值(ElementType)有：`CONSTRUCTOR`、`FIELD`、`LOCAL_VARIABLE`、`METHOD`、`PACKAGE`、`PARAMETER`、`TYPE`。

```java
@Target(ElementType.TYPE)
public @interface Table {
/**
  * 数据表名称注解，默认值为类名称
  * @return
  */
    public String tableName() default "className";
}

@Target(ElementType.FIELD)
public @interface NoDBColumn {

}
```
注解Table 可以用于注解类、接口(包括注解类型) 或enum声明,而注解NoDBColumn仅可用于注解类的成员变量。

## @Retention
@Retention定义了该Annotation被保留的时间长短：某些Annotation仅出现在源代码中，而被编译器丢弃；而另一些却被编译在class文件中；编译在class文件中的Annotation可能会被虚拟机忽略，而另一些在class被装载时将被读取。
> **请注意并不影响class的执行，因为Annotation与class在使用上是被分离的。**

@Retention的取值（RetentionPoicy）有：
- SOURCE：在源文件中有效（即源文件保留）

- CLASS：在class文件中有效（即class保留）

- RUNTIME：在运行时有效（即运行时保留）

Retention meta-annotation类型有唯一的value作为成员，它的取值来自java.lang.annotation.RetentionPolicy的枚举类型值。具体实例如下：

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Column {
    public String name() default "fieldName";
    public String setFuncName() default "setField";
    public String getFuncName() default "getField";
    public boolean defaultDBValue() default false;
}
```
Column注解的的RetentionPolicy的属性值是`RUNTIME`，这样注解处理器可以通过反射，获取到该注解的属性值，从而去做一些运行时的逻辑处理

## @Documented
@Documented用于描述其它类型的annotation应该被作为被标注的程序成员的公共API，因此可以被例如javadoc此类的工具文档化。Documented是一个标记注解，没有成员。

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Column {
    public String name() default "fieldName";
    public String setFuncName() default "setField";
    public String getFuncName() default "getField"; 
    public boolean defaultDBValue() default false;
}
```

## @Inherited
@Inherited 元注解是一个标记注解，@Inherited阐述了某个被标注的类型是被继承的。如果一个使用了@Inherited修饰的annotation类型被用于一个class，则这个annotation将被用于该class的子类。

> **注意：@Inherited annotation类型是被标注过的class的子类所继承。类并不从它所实现的接口继承annotation，方法并不从它所重载的方法继承annotation。**

当@Inherited annotation类型标注的annotation的Retention是RetentionPolicy.RUNTIME，则反射API增强了这种继承性。如果我们使用java.lang.reflect去查询一个@Inherited annotation类型的annotation时，反射代码检查将展开工作：检查class和其父类，直到发现指定的annotation类型被发现，或者到达类继承结构的顶层。

实例代码：
```java
@Inherited
public @interface Greeting {
    public enum FontColor{ BULE,RED,GREEN};
    String name();
    FontColor fontColor() default FontColor.GREEN;
}
```

# 自定义注解
使用`@interface`自定义注解时，自动继承了`java.lang.annotation.Annotation`接口，由编译程序自动完成其他细节。在定义注解时，不能继承其他的注解或接口。`@interface`用来声明一个注解，其中的每一个方法实际上是声明了一个配置参数。方法的名称就是参数的名称，返回值类型就是参数的类型（返回值类型只能是基本类型、Class、String、enum）。可以通过default来声明参数的默认值。

定义注解格式：
```java
public @interface Annotation_name {body}
```
注解参数的可支持数据类型：
1. 所有基本数据类型（int,float,boolean,char等)
2. String类型
3. Class类型
4. enum类型
5. Annotation类型
6. 以上所有类型的数组

## Annotation参数的设定
> 1. 只能用public或默认(default)这两个访问权修饰。例如，String value();这里把方法设为defaul默认类型。

> 2. 参数成员只能用基本数据类型和 String,Enum,Class,annotations等数据类型，以及这一些类型的数组。例如`String value();`的，这里的参数成员就为String。

> 3. 如果只有一个参数成员，最好把参数名称设为"value"，后加小括号。例:下面的例子FruitName注解就只有一个参数成员。

简单的自定义注解和使用注解实例：
```java
import java.lang.annotation.*;

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Description {
    String desc();
    String author();
    int age() default 18;
}
```
