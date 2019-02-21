---
title: Spring整合Druid连接池错误
date: 2017-07-03 01:00:00
tags: [freemarker,druid,spring]
categories:
- framework
- Spring
---

最近重新Review之前的一个项目，使用的是Spring整合Druid作为连接池。Maven重新打包后出现异常：
```bash
Caused by: java.sql.SQLException: Access denied for user 'silverlining'@'localhost' (using password: YES)
```
感觉很奇怪，明明配置文件中properties的username属性为root，竟然会变成我的电脑名。

一番Google后终于明白，原来是因为我在properties的数据库用户名属性名用了username，被Spring引入为`${username}`。而项目中数据库的配置方式是`<context:property-placeholder>`。这里的`system-properties-mode`没有设置，默认值`environment`优先匹配系统属性，再从location文件中匹配属性。这里它优先找到系统的`username`属性，也就解释了为什么连接数据库的用户一直是电脑用户名了。

<!-- more -->

我们来仔细看看`system-properties-mode`属性：
- ENVIRONMENT -indicates placeholders should be resolved against the current Environment and against any local properties;（默认，优先系统属性和任何本地属性）

- NEVER -indicates placeholders should be resolved only against local properties and never against system properties;（只使用本地配置文件）

- FALLBACK -indicates placeholders should be resolved against any local properties and then against system properties;（先本地配置文件再系统属性）

- OVERRIDE -indicates placeholders should be resolved first against system properties and then against any local properties;（先系统属性再本地配置文件）

了解了问题所在，就很好解决了，修改`context:property-placeholder`属性：
```xml
<context:property-placeholder location="classpath*:jdbc.properties" system-properties-mode="FALLBACK"/>
```

或者采用另一种导入配置文件的方式：
```xml
<bean id="propertyConfigurer"
		class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
		<property name="location" value="classpath:jdbc.properties" />
		<property name="systemPropertiesMode" value="1"/>
</bean>
```
这种方法导入配置文件不会有上面的问题，在`PropertyPlaceholderConfigurer`类的注释中可以看到，它只有3种配置方式，并且默认就是`FALLBACK`：
```java
/**
  * Set how to check system properties: as fallback, as override, or never.
  * For example, will resolve ${user.dir} to the "user.dir" system property.
  * <p>The default is "fallback": If not being able to resolve a placeholder
  * with the specified properties, a system property will be tried.
  * "override" will check for a system property first, before trying the
  * specified properties. "never" will not check system properties at all.
  * @see #SYSTEM_PROPERTIES_MODE_NEVER
  * @see #SYSTEM_PROPERTIES_MODE_FALLBACK
  * @see #SYSTEM_PROPERTIES_MODE_OVERRIDE
  * @see #setSystemPropertiesModeName
  */
public void setSystemPropertiesMode(int systemPropertiesMode) {
	this.systemPropertiesMode = systemPropertiesMode;
}
```
3种模式对应的值：

|    Modifier and Type    |         Constant Field          | Value |
| ----------------------- | ------------------------------- | ----- |
| public static final int | SYSTEM_PROPERTIES_MODE_FALLBACK | 1     |
| public static final int | SYSTEM_PROPERTIES_MODE_NEVER    | 0     |
| public static final int | SYSTEM_PROPERTIES_MODE_OVERRIDE | 2     |

一点感想：这个问题前前后后折腾了好久，归结起来其实还是文档看的不够仔细。如果在使用`propertyConfigurer`这个配置的时候注意一下默认值，就不会出现这样的问题了。以后要养成仔细阅读文档的好习惯。
