---
title: SpringCloud 服务发现
date: 2019-04-20 16:49:10
tags: SpringCloud
categories: SpringCloud系列
---

添加依赖
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

<!-- more -->

添加 `@EnableEurekaClient` 注解，开启服务发现
```java
@EnableEurekaClient
@SpringBootApplication
public class SpringCloudServiceOneApplication {
    public static void main(String[] args) {
        SpringApplication.run(SpringCloudServiceOneApplication.class, args);
    }
}
```

开启服务注册配置，并设置注册中心的地址
```properties
server.port=10001
spring.application.name=spring-cloud-service-one-example

eureka.client.fetch-registry=true
eureka.client.register-with-eureka=true
eureka.client.service-url.defaultZone=http://localhost:8080/eureka
```
依次打包并启动 `spring-cloud-eureka-example` `spring-cloud-service-one-example` 项目，查看输出的日志:
```log
2019-04-20 16:58:46.270  INFO 55738 --- [nio-8080-exec-2] c.n.e.registry.AbstractInstanceRegistry  : Registered instance SPRING-CLOUD-SERVICE-ONE-EXAMPLE/localhost:spring-cloud-service-one-example:10001 with status UP (replication=false)
```
```log
2019-04-20 16:58:46.272  INFO 55809 --- [nfoReplicator-0] com.netflix.discovery.DiscoveryClient    : DiscoveryClient_SPRING-CLOUD-SERVICE-ONE-EXAMPLE/localhost:spring-cloud-service-one-example:10001 - registration status: 204
2019-04-20 16:59:16.137  INFO 55809 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Disable delta property : false
2019-04-20 16:59:16.137  INFO 55809 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Single vip registry refresh property : null
2019-04-20 16:59:16.137  INFO 55809 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Force full registry fetch : false
2019-04-20 16:59:16.137  INFO 55809 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Application is null : false
2019-04-20 16:59:16.137  INFO 55809 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Registered Applications size is zero : true
2019-04-20 16:59:16.137  INFO 55809 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Application version is -1: false
2019-04-20 16:59:16.137  INFO 55809 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : Getting all instance registry info from the eureka server
2019-04-20 16:59:16.161  INFO 55809 --- [freshExecutor-0] com.netflix.discovery.DiscoveryClient    : The response status is 200
```
通过以上日志能够查看出服务注册发现是正常运行的，访问 http://localhost:8080 查看注册服务的状态。

代码案例：[https://github.com/JiangYongKang/spring-cloud-examples](https://github.com/JiangYongKang/spring-cloud-examples)
