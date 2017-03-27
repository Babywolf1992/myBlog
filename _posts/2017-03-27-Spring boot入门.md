---
layout: post
title:  Spring Boot入门
date:   2017-03-27 23:01:00 +0800
tags: 能工巧匠集
---

## Spring Boot介绍

Spring Boot简化了基于Spring的应用开发，你只需要"run"就能创建一个独立的，产品级别的Spring应用。

#### 该项目主要的目的是：

- 为所有Spring开发提供一个从根本上更快，且随处可得的入门体验。
- 开箱即用，但通过不采用默认设置可以快速摆脱这种方式。
- 提供一系列大型项目常用的非功能性特征，比如：内嵌服务器，安全，指标，健康检测，外部化配置。
- 绝对没有代码生成，也不需要XML配置。

#### 系统要求:

建议使用java7以上,maven(3.2+)或Gradle(1.12+)。

#### 快速使用:

pom.xml
```java
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>1.5.1.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
    </dependencies>
    
```

快速启动:
```java
@EnableAutoConfiguration
@RestController
public class Application {
    @RequestMapping(value = "/")
    String home() {
        return "hello world";
    }

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

运行后console:(注意这里没有任何配置,默认8080端口)
![](/assets/images/2017/spring-boot启动-1.png)
浏览器打开查看:
![](/assets/images/2017/spring-boot启动-2.png)
到此一个简易的Spring Boot项目已经完成了

想要修改端口号,需要新建配置文件,在resources目录下添加application.properties
```java
server.port=9000
```
好了,再重新运行看看。

## 最后

关于Spring Boot,我也是处在学习当中,希望和大家共同进步。