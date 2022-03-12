---
title: SpringBoot（一）：入门体验
date: 2020-11-11 12:56:29
tags: SpringBoot
categories: SpringBoot 系列
---

**SpringBoot** 是 **Spring** 框架的进一步封装，其设计的目的是用来简化 Spring 应用的框架搭建和开发过程。SpringBoot 使用 **约定大于配置** 的原则，使得开发人员不需要在进行大量样板化的配置，从而专注与业务逻辑上的开发工作。本文通过一个 "Hello World" 示例程序，展示 SpringBoot 在依赖自动管理，自动配置带来的快速开发体验。

<!-- more -->

### 添加依赖

SpringBoot 支持自动依赖关系的配置，每一个 SpringBoot 版本都会在它的 POM 文件中提供它所支持的依赖列表和版本。因此，使用自动依赖关系配置可以避免不同版本的依赖包冲突的问题，在升级 SpringBoot 版本时也会一致的更新所有依赖项。这里我们通过继承 `SpringBoot POM` 的方式创建项目。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <groupId>org.spring.boot.examples</groupId>
    <artifactId>spring-boot-examples</artifactId>
    <packaging>pom</packaging>
    <version>1.0-SNAPSHOT</version>
    <modelVersion>4.0.0</modelVersion>
    <description>SpringBoot examples</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.5.RELEASE</version>
        <relativePath/>
    </parent>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>
```

### 创建启动类

创建一个 `SpringBootRestful` 类，并添加以下代码：

```java
@RestController
@SpringBootApplication
public class SpringBootRestful {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootRestful.class, args);
    }

    @GetMapping
    public ResponseEntity<?> hello() {
        return ResponseEntity.ok("Hello World");
    }

}
```

`@SpringBootApplication` 注解用于开启自动配置和组件扫描，它包含了下面三个注解的功能。

- `@EnableAutoConfiguration` 启用 SpringBoot 的自动配置功能
- `@ComponentScan` 对当前类所在的包开启组件扫描功能
- `@SpringBootConfiguration` 继承自 `@Configuration`，二者功能也一致，标注当前类是配置类。

### 访问接口

启动 `SpringBootRestful`，打开浏览器访问 [http://localhost:8080](http://localhost:8080)，如果能看到浏览器上显示 "Hello World"，一个基本的 SpringBoot 框架就搭建完毕了。