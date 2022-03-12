---
title: SpringCloud 服务注册
date: 2019-04-20 12:50:30
tags: SpringCloud
categories: SpringCloud系列
---

添加依赖
```xml

<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
    </dependency>
</dependencies>
```

<!-- more -->

启动类上添加 `@EnableEurekaServer` 注解开启服务发现
```java
@EnableEurekaServer
@SpringBootApplication
public class SpringCloudEurekaApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudEurekaApplication.class, args);
    }
}
```

添加下面的依赖并禁止注册中心的自我注册
```properties
server.port=8080
spring.application.name=spring-cloud-eureka-example

eureka.client.fetch-registry=false
eureka.client.register-with-eureka=false
eureka.instance.hostname=localhost
eureka.client.serviceUrl.defaultZone=http://${eureka.instance.hostname}:${server.port}/eureka
```
这里会有两个地址。第一个 http://localhost:8080 是注册中心默认的管理页面，上面可以便捷的查看服务的注册情况。第二个 http://localhost:8080/eureka 是注册中心的注册地址，其他服务需要在注册中心注册需要配置这个地址。

代码案例：[https://github.com/JiangYongKang/spring-cloud-examples](https://github.com/JiangYongKang/spring-cloud-examples)
