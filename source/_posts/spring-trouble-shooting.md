---
title: sqoop-trouble-shooting
date: 2016-10-14 17:37:51
toc: true
tags:
- spring spring-data hadoop
categories: 
- spring
---

记录使用Spring过程中遇到的问题、原因及解决办法。

# NoClassDefFoundError: org/apache/hadoop/conf/Configuration$DeprecationDelta

使用spring-data-hadoop-hbase时遇到如下错误：
`Exception in thread "main" java.lang.NoClassDefFoundError: org/apache/hadoop/conf/Configuration$DeprecationDelta`

原因：spring-data-hadoop-hbase版本为2.2.0，其中使用的hadoop-hdfs版本为2.6.0，因此出现如上错误。

解决办法：使用同版本的hadoop包。

PS:若遇到错误：org.eclipse.jdt.internal.compiler.CompilationResult.getProblems，这是由于jsp解析包依赖冲突所致，将其从hadoop/hive/hbase等相关的包中exclude掉即可。

# java.lang.NoSuchMethodError: com.fasterxml.jackson.databind.ObjectWriter.forType

在一个使用了spring boot的项目中，使用spark mllib加载model时，遇到如下错误：
```
Caused by: java.lang.NoSuchMethodError: com.fasterxml.jackson.databind.ObjectWriter.forType(Lcom/fasterxml/jackson/databind/JavaType;)Lcom/fasterxml/jackson/databind/ObjectWriter;
```
从而导致无法load model；

`由于spark 1.6与spring中依赖的jackson包冲突，在pom.xml中增加依赖版本2.4.4，问题解决。`

更正：
上述方法未能解决问题，由于是在idea中进行调试，在修改pom.xml，增加2.4.4版本依赖后，在tomcat目录下存在两个版本的jar包，因此不会报错。
但是打包的时候，只会取2.4.4版本的JAR包，从而导致打包之后的war包发布时出错。
进一步解决办法，在pom.xml中添加如下配置：

```
<dependency>
    <groupId>com.fasterxml.jackson.module</groupId>
    <artifactId>jackson-module-scala_${scala.version}</artifactId>
    <version>2.8.5</version>
    <scope>compile</scope>
    <exclusions>
        <exclusion>
            <artifactId>guava</artifactId>
            <groupId>com.google.guava</groupId>
        </exclusion>
    </exclusions>
</dependency>
```

# BeanCreationException Error creating bean with name 'dataSource'

spring boot的项目无法启动，异常信息如下：

```
Caused by: org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'dataSource' defined in class path resource [org/springframework/boot/autoconfigure/jdbc/DataSourceConfiguration$Tomcat.class]: Bean instantiation via factory method failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.apache.tomcat.jdbc.pool.DataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Cannot determine embedded database driver class for database type NONE. If you want an embedded database please put a supported one on the classpath. If you have database settings to be loaded from a particular profile you may need to active it (no profiles are currently active).
	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:599)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.instantiateUsingFactoryMethod(AbstractAutowireCapableBeanFactory.java:1128)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBeanInstance(AbstractAutowireCapableBeanFactory.java:1022)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.doCreateBean(AbstractAutowireCapableBeanFactory.java:512)
	at org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory.createBean(AbstractAutowireCapableBeanFactory.java:482)
	at org.springframework.beans.factory.support.AbstractBeanFactory$1.getObject(AbstractBeanFactory.java:306)
	at org.springframework.beans.factory.support.DefaultSingletonBeanRegistry.getSingleton(DefaultSingletonBeanRegistry.java:230)
	at org.springframework.beans.factory.support.AbstractBeanFactory.doGetBean(AbstractBeanFactory.java:302)
	at org.springframework.beans.factory.support.AbstractBeanFactory.getBean(AbstractBeanFactory.java:197)
	at org.springframework.beans.factory.support.DefaultListableBeanFactory.preInstantiateSingletons(DefaultListableBeanFactory.java:754)
	at org.springframework.context.support.AbstractApplicationContext.finishBeanFactoryInitialization(AbstractApplicationContext.java:866)
	at org.springframework.context.support.AbstractApplicationContext.__refresh(AbstractApplicationContext.java:542)
	at org.springframework.context.support.AbstractApplicationContext.jrLockAndRefresh(AbstractApplicationContext.java)
	at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java)
	at org.springframework.boot.context.embedded.EmbeddedWebApplicationContext.refresh(EmbeddedWebApplicationContext.java:122)
	at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:761)
	at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:371)
	at org.springframework.boot.SpringApplication.run(SpringApplication.java:315)
	at org.springframework.boot.web.support.SpringBootServletInitializer.run(SpringBootServletInitializer.java:151)
	at org.springframework.boot.web.support.SpringBootServletInitializer.createRootApplicationContext(SpringBootServletInitializer.java:131)
	at org.springframework.boot.web.support.SpringBootServletInitializer.onStartup(SpringBootServletInitializer.java:86)
	at org.springframework.web.SpringServletContainerInitializer.onStartup(SpringServletContainerInitializer.java:169)
	at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5604)
	at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:147)
	... 42 more
Caused by: org.springframework.beans.BeanInstantiationException: Failed to instantiate [org.apache.tomcat.jdbc.pool.DataSource]: Factory method 'dataSource' threw exception; nested exception is org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Cannot determine embedded database driver class for database type NONE. If you want an embedded database please put a supported one on the classpath. If you have database settings to be loaded from a particular profile you may need to active it (no profiles are currently active).
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:189)
	at org.springframework.beans.factory.support.ConstructorResolver.instantiateUsingFactoryMethod(ConstructorResolver.java:588)
	... 65 more
Caused by: org.springframework.boot.autoconfigure.jdbc.DataSourceProperties$DataSourceBeanCreationException: Cannot determine embedded database driver class for database type NONE. If you want an embedded database please put a supported one on the classpath. If you have database settings to be loaded from a particular profile you may need to active it (no profiles are currently active).
	at org.springframework.boot.autoconfigure.jdbc.DataSourceProperties.determineDriverClassName(DataSourceProperties.java:245)
	at org.springframework.boot.autoconfigure.jdbc.DataSourceProperties.initializeDataSourceBuilder(DataSourceProperties.java:182)
	at org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration.createDataSource(DataSourceConfiguration.java:42)
	at org.springframework.boot.autoconfigure.jdbc.DataSourceConfiguration$Tomcat.dataSource(DataSourceConfiguration.java:53)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.springframework.beans.factory.support.SimpleInstantiationStrategy.instantiate(SimpleInstantiationStrategy.java:162)
	... 66 more
```

由于没有使用数据库，从而导致spring在自动加载dataSource时报错，修改项目中spring相关的配置类如下：
修改Application类中的注解：
原注解：
//@SpringBootApplication
修改为：
@Configuration
@ComponentScan
修改Spring配置类：
增加注解：@EnableAutoConfiguration(exclude = {DataSourceAutoConfiguration.class, DataSourceTransactionManagerAutoConfiguration.class, HibernateJpaAutoConfiguration.class})

[参考链接](http://stackoverflow.com/questions/43136602/error-creating-bean-with-name-datasource-defined-in-class-path-resource-spri)

[参考链接](http://stackoverflow.com/questions/36387265/disable-all-database-related-auto-configuration-in-spring-boot)

[参考链接](http://docs.spring.io/spring-boot/docs/current-SNAPSHOT/reference/htmlsingle/#auto-configuration-classes)

