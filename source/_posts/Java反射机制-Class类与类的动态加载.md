---
title: Java反射机制-Class类与类的动态加载
date: 2016-05-05 14:47:28
tags: [Java,reflection]
categories:
- Java
- reflection
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

- - -

# Class对象
Java是面向对象的，除了Java静态内容，基本数据类型也使用类进行了封装。那么所有这些对象又是谁的对象呢？Java中`java.lang.Class`类就是所有对象的类。

在`java.lang.Class`中有如下构造函数。其中明确Class类是私有的，只能由JVM创建。
```java
/*
 * Private constructor. Only the Java Virtual Machine creates Class objects.
 * This constructor is not used and prevents the default constructor being generated.
 */
private Class(ClassLoader loader) {
	// Initialize final field for classLoader.  The initialization value of non-null
	// prevents future JIT optimizations from assuming this final field is null.
	classLoader = loader;
}
```

<!-- more -->

# 获取Class对象的方法
获取Class对象共有四种方法：
```java
Foo foo1 = new Foo();

//第一种表示方式
//任何一个类都有一个隐含的静态成员变量class
Class c1 = Foo.class;
//第二中表示方式
Class c2 = foo1.getClass();
//一个类只可能是Class类的一个实例对象
System.out.println(c1 == c2);

//第三种表达方式,实现类的动态加载
Class c3 = null;
try {
	c3 = Class.forName("com.sunnywr.reflect.Foo");
} catch (ClassNotFoundException e) {
	e.printStackTrace();
}
System.out.println(c2==c3);

//第四种方式
//通过类的类类型(Class type)创建该类的对象实例
try {
	Foo foo = (Foo)c1.newInstance();//需要有无参数的构造方法
	foo.print();
} catch (InstantiationException e) {
	e.printStackTrace();
} catch (IllegalAccessException e) {
	e.printStackTrace();
}
```

# Class类的动态加载
## 为什么需要动态加载
我们首先考虑以下代码`Office.java`：
```java
class Office {
	public static void main(String[] args) {
		if("Word".equals(args[0])) {
			Word word = new Word();
			word.start();
		}
		if("Excel".equals(args[0])) {
			Excel excel = new Excel();
			excel.start();
		}
	}
}
```
显然直接编译这个文件会报4个错误，因为`Word`，`Excel`类和两个`start()`方法未定义。可是`Word`和`Excel`类一定都要存在吗？如果我们只需要用`Word`，并且`Word`类也存在，可是由于缺少`Excel`类导致`Office`不能通过编译与使用。这时我们就需要动态加载Class类。

## Class的动态加载
先给出`Word`类的实现：
```java
class Word {
	public static void start() {
		System.out.println("Word starts...");
	}
}
```
这里假如我们要实现100个功能，有1个有问题，其他99个都用不了。这显然不是我们想达到的目的。因此我们要对`Office`实现Class的动态加载：
```java
class OfficeBetter {
	public static void main(String[] args) {
		try {
			//动态加载类
			Class c = Class.forName(args[0]);
			Word word = (Word)c.newInstance();
			word.start();
		} catch(Exception e) {
			e.printStackTrace();
		}
	}
}
```

当我们执行`Word`时，不报错；执行`Excel`时由于找不到`Excel`类则会报错：
```bash
$ javac OfficeBetter.java
$ java OfficeBetter Word
Word starts...
```

## 接口的封装
细心的读者应该已经发现以上程序的问题了。在我们对一个类进行了`Class.newInstance()`实例化以后，需要强制类型转化成一个具体的类类型（Class type)，这里我们用了`Word`进行类型转化。但是这样的代码对于`Excel`类就会有问题。
我们当然可以像最开始那样使用if条件判断，但是这样带来的问题就是每次我们添加一个功能都需要重新更改`OfficeBetter`的代码和重新编译。其实我们只需要定义一个接口类， 让各个子模块实现接口类即可实现动态加载，而且不需要改变`Office`类的实现。

```java
interface OfficeService {
	public void start();
}
```

```java
class Word implements OfficeService {
	public void start() {
		System.out.println("Word starts...");
	}
}
```

```java
class OfficeBetter {
	public static void main(String[] args) {
		try {
			//动态加载类
			Class c = Class.forName(args[0]);
			OfficeService os = (OfficeService)c.newInstance();
			os.start();
		} catch(Exception e) {
			e.printStackTrace();
		}
	}
}
```

这样就完成了动态加载的过程。重新编译几个类，然后执行`java OfficeBetter Word`将会调用`Word`类中的`start()`方法，与`Excel`类无关。
