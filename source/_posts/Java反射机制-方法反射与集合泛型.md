---
title: Java反射机制-方法反射与集合泛型
date: 2016-05-06 19:26:28
tags: [Java,reflection]
categories:
- Java
- reflection
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

> 测试代码：
> https://github.com/njZhuMin/JavaDemo/tree/master/reflect/ReflectMethod.java

- - -

# 如何获取方法
`方法的名称`和`方法的参数列表`才能唯一确定某个方法。每一个对象都隐含有`invoke()`方法，来实现反射操作：
```java
method.invoke(object, params);
```
而需要获取一个方法，首先需要获取类的信息。获取类的信息又需要先获取类的类类型。一般获取方法的实现为：
```java
A a = new A();
Method m = a.getClass().getMethod(method_name, param_list);
```

<!-- more -->

# 方法的反射
例如我们定义一个类`A`，`A`中有三个`print`方法：
```java
class A {
    public void print(int a, int b) {
        System.out.println(a + b);
    }

    public void print(String a, String b) {
        System.out.println(a.toUpperCase() + "," + b.toLowerCase());
    }

    public void print() {
        System.out.println("Hello world!");
    }
}
```

以下展示方法的反射方法：
```java
public class ReflectMethod {
    public static void main(String[] args) {
        //获取print(int, int)方法
        A a1 = new A();
        //先获取类类型
        Class c = a1.getClass();
        //获取方法，方法名与参数列表
        try {
            //Method m = c.getMethod("print", new Class[]{int.class, int.class});
            Method m1 = c.getMethod("print", int.class, int.class);
            //反射print()方法
            //Object o = m1.invoke(a1, new Object[]{10, 20});
            Object o1 = m1.invoke(a1, 10, 20);
            System.out.println("==========================");

            Method m2 = c.getMethod("print", String.class, String.class);
            //a1.print("hello", "WORLD");
            Object o2 = m2.invoke(a1, "hello", "WORLD");
            System.out.println("==========================");

            Method m3 = c.getMethod("print");
            Object o3 = m3.invoke(a1);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

# 泛型的本质
这里用集合来举例：
```java
//定义一个集合
ArrayList list1 = new ArrayList();
//定义一个泛型集合
ArrayList<String> list2 = new ArrayList<String>();
```
对于泛型集合`list2`，由于泛型的约束，我们只能向其中添加`String`对象：
```java
list2.add("hello");	//OK
list2.add(20); 		//Error
```
比较`list1`和`list2`的类类型：
```java
//return true
System.out.println(list1.getClass() == list2.getClass());
```

为什么会出现这样的结果呢？因为对于方法的反射都是发生在编译之后的。这里`list1`与`list2`的类类型相同，说明编译之后的集合是`去泛型化`的。

# 通过反射绕过编译
> 在Java中，集合的泛型用于检查输入类型的错误，只在编译阶段有效。绕过编译阶段泛型就无效了。

```java
public class ReflectMethod {
    public static void main(String[] args) {
        ArrayList list1 = new ArrayList();
        ArrayList<String> list2 = new ArrayList<String>();
        System.out.println(list1.getClass() == list2.getClass());

        list2.add("hello");
        try {
            Method m = list2.getClass().getMethod("add", Object.class);
            m.invoke(list2, 100);
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            System.out.println(list2.size());
        }
    }
}
```
可以看到，此时可以成功的向`ArrayList<String> list2`中添加整型100，且list2的大小为2。
> 注意，这时不能再用`for each`方法进行集合遍历，会抛出类型转化异常。
