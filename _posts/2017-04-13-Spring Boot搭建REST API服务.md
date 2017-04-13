---
layout: post
title:  Spring Boot搭建REST API服务
date:   2017-04-13 21:12:00 +0800
tags: 能工巧匠集
---

REST指的是Representational State Transfer。我对它的理解就是讲网站的API接口已资源的形式表现出来，即REST API（想要更深入理解的同学可以自己查阅相关资料）。

使用Spring Boot搭建REST API Service。首先使用maven新建项目。添加相应依赖。

pom.xml

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
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <scope>test</scope>
            </dependency>
        </dependencies>

然后是应用程序启动类

    @SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            SpringApplication.run(Application.class, args);
        }
    }
    

简单编写了一个entity，User类，代码也很简单

    public class User {
        int id;
    
        String name;
    
        public int getId() {
            return id;
        }
    
        public void setId(int id) {
            this.id = id;
        }
    
        public String getName() {
            return name;
        }
    
        public void setName(String name) {
            this.name = name;
        }
    }
    

下面是重点Controller的编写

    @RestController
    @RequestMapping(value = "/index")
    public class IndexController {
        @RequestMapping
        public String index() {
            return "welcome";
        }
    
        @RequestMapping(value = "/get")
        public HashMap<String, Object> get(@RequestParam String name) {
            HashMap<String, Object> map = new HashMap<String, Object>();
            map.put("title", "hello");
            map.put("name", name);
            return map;
        }
    
        @RequestMapping(value = "/get/{id}/{name}")
        public User getUser(@PathVariable int id, @PathVariable String name) {
            User user = new User();
            user.setId(id);
            user.setName(name);
            return user;
        }
    }
    

@RequestParam 可以对请求的参数进行绑定，@PathVariable可以动态获取url中的数据，大家可以自己动手写一写，就明白其中含义。

测试
![](/assets/images/2017/新光村-1.jpg)