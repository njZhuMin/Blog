---
title: Java注解机制-自定义注解
date: 2016-05-08 10:46:13
tags: [Java,annotation]
categories:
- Java
- annotation
---

> Java系列文章：
> http://zhum.in/blog/categories/Java/

> 测试代码：
> https://github.com/njZhuMin/JavaDemo/tree/master/annotation/ParseAnn

- - -

# 解析注解
解析注解是通过反射来获取类、方法或成员上的`运行时`注解信息，从而实现动态控制程序运行的逻辑。

## 注解处理器类库
Java使用Annotation接口来代表程序元素前面的注解，该接口是所有Annotation类型的父接口。除此之外，Java在`java.lang.reflect`包下新增了`AnnotatedElement`接口，该接口代表程序中可以接受注解的程序元素，该接口主要有如下几个实现类：
- Class：类定义
- Constructor：构造器定义
- Field：累的成员变量定义
- Method：类的方法定义
- Package：类的包定义

<!-- more -->

java.lang.reflect 包下主要包含一些实现反射功能的工具类，实际上，java.lang.reflect 包所有提供的反射API扩充了读取运行时Annotation信息的能力。当一个Annotation类型被定义为运行时的Annotation后，该注解才能是运行时可见，当class文件被装载时被保存在class文件中的Annotation才会被虚拟机读取。

AnnotatedElement 接口是所有程序元素（Class、Method和Constructor）的父接口，所以程序通过反射获取了某个类的AnnotatedElement对象之后，程序就可以调用该对象的如下四个个方法来访问Annotation信息：

> 1. 返回改程序元素上存在的、指定类型的注解，如果该类型注解不存在，则返回null。
> ```java
> <T extends Annotation> T getAnnotation(Class<T> annotationClass)`;
> ```

> 2. 返回该程序元素上存在的所有注解。
> ```java
> Annotation[] getAnnotations();
> ```

> 3. 判断该程序元素上是否包含指定类型的注解，存在则返回true，否则返回false。
> ```java
> boolean is AnnotationPresent(Class<?extends Annotation> annotationClass);
> ```

> 4. 返回直接存在于此元素上的所有注释。与此接口中的其他方法不同，该方法将忽略继承的注释（如果没有注释直接存在于此元素上，则返回长度为零的一个数组）。该方法的调用者可以随意修改返回的数组，这不会对其他调用者返回的数组产生任何影响。
> ```java
> Annotation[] getDeclaredAnnotations();
> ```

## 注解解析获取实现
```java
package com.sunnywr;
public interface Person {
    public String name();
    public int age();
    @Deprecated
    public void sing();
}
```

```java
package com.sunnywr;

@Description("I am class annotation")
public class Child implements Person {
    @Override
    @Description("I am method annotation")
    public String name() {
        return null;
    }

    @Override
    public int age() {
        return 0;
    }

    @Override
    public void sing() {}
}
```

```java
package com.sunnywr;
import java.lang.annotation.*;

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Description {
    String value();
}
```

```java
package com.sunnywr;
import java.lang.annotation.Annotation;
import java.lang.reflect.Method;

public class ParseAnn {
    public static void main(String[] args) {
        //第一种解析方法
        try {
            //使用类加载器加载类
            Class c = Class.forName("com.sunnywr.Child");
            //获取类的注解
            if(c.isAnnotationPresent(Description.class)) {
                //获取注解实例
                Description d = (Description)c.getAnnotation(Description.class);
                System.out.println(d.value());
            }

            //获取方法的注解
            Method[] methods = c.getMethods();
            for(Method m : methods) {
                if(m.isAnnotationPresent(Description.class)) {
                    Description d = (Description)m.getAnnotation(Description.class);
                    System.out.println(d.value());
                }
            }

            //第二种解析方法
            for(Method m : methods) {
                Annotation[] annotations = m.getAnnotations();
                for(Annotation a : annotations) {
                    if(a instanceof Description) {
                        Description d = (Description)a;
                        System.out.println(d.value());
                    }
                }
            }
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

# 注解的继承
## 注解的继承测试
```java
package com.sunnywr;

@Description("I am interface")
public interface Person {
    @Description("I am interface method")
    public String name();
    public int age();
    @Deprecated
    public void sing();
}
```

```java
package com.sunnywr;

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
    public void sing() { }
}
```

运行`ParseAnn`后结果无输出，也就是注解并没有继承成功。那么问题出在哪儿呢？

## 注解的继承机制
我们接着去看`@Inherited`的注释。在`java.lang.annotation.Inherited`中有如下解释：
> Indicates that an annotation type is automatically inherited. If an Inherited meta-annotation is present on an annotation type declaration, and the user queries the annotation type on a class declaration, and the class declaration has no annotation for this type, then the class's superclass will automatically be queried for the annotation type. This process will be repeated until an annotation for this type is found, or the top of the class hierarchy (Object) is reached. If no superclass has an annotation for this type, then the query will indicate that the class in question has no such annotation.

> Note that this meta-annotation type has no effect if the annotated type is used to annotate anything other than a class. Note also that this meta-annotation only causes annotations to be inherited from superclasses; annotations on implemented interfaces have no effect.

我们可以看到，`@Inherited`的关键字只作用于`Class`级别的注解，并且只对子类有效。注解和接口都是无效的。因此我们尝试改进`Person`接口为父类。

## 注解继承的改进
```java
package com.sunnywr;

@Description("I am interface")
public class Person {
    @Description("I am interface method")
    public String name() {
        return null;
    }
    public int age() {
        return 0;
    }
    @Deprecated
    public void sing() {
        System.out.println("Singing...");
    }
}
```

```java
package com.sunnywr;

public class Child extends Person {
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
        System.out.println("Singing...");
    }
}
```
再次运行`ParseAnn`后输出`I am interface`，即成功继承了父类的Class注解。
