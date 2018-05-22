---
title: maven-trouble-assembly-shade
date: 2016-09-04 00:00:00
toc: true
tags:
- maven trouble
categories:
- maven
---

# 问题

使用maven assembly打包后，启动时报错如下：

```
Exception in thread "main" org.springframework.beans.factory.parsing.BeanDefinitionParsingException: Configuration problem: Unable to locate Spring NamespaceHandler for XML schema namespace [http://www.springframework.org/schema/context]
```

# 原因

这是 assembly 插件的一个 bug：[http://jira.codehaus.org/browse/MASSEMBLY-360](https://link.jianshu.com?t=http://jira.codehaus.org/browse/MASSEMBLY-360)，它在对第三方打包时，对于 META-INF 下的 spring.handlers，spring.schemas 等多个同名文件进行了覆盖，遗漏掉了一些版本的 xsd 本地映射。

# 解决办法

改用maven-shade-plugin来打包。

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <filters>
                    <filter>
                        <artifact>*:*</artifact>
                        <excludes>
                            <exclude>META-INF/*.SF</exclude>
                            <exclude>META-INF/*.DSA</exclude>
                            <exclude>META-INF/*.RSA</exclude>
                        </excludes>
                    </filter>
                </filters>
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/spring.handlers</resource>
                    </transformer>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.fxc.rpc.impl.member.MemberProvider</mainClass>
                    </transformer>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/spring.schemas</resource>
                    </transformer>
                </transformers>
            </configuration>
        </execution>
    </executions>
</plugin>
```



