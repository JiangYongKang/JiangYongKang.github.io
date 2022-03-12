---
title: SpringBoot & Maven 引用本地依赖文件并打包
date: 2021-10-24 06:36:32
tags: ["SpringBoot", "Maven"]
categories: 服务端开发
---

在开发项目的过程中，有可能会遇到需要将本地依赖打包到 Maven 项目中的情况。如果使用的是 IntelliJ IDEA 或者 Eclipse 可以将本地依赖添加进去，这样在本地开发环境是没有问题的。但是如果需要打包并且部署的情况下，就会出现找不到类的情况。

<!-- more -->

### 0x00 拷贝依赖

在 `src/main/resources/` 目录下新建 `libraries` 文件夹并将本地依赖拷贝进去。

### 0x01 本地开发

引用本地依赖，这里用 MySQL 驱动文件举例。

```
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.26</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/src/main/resources/libraries/mysql-connector-java.jar</systemPath>
</dependency>
```

完成这一步后，本地开发是可以正常使用的，但是部署到服务器仍然会出现类加载失败的错误。所以我们需要进一步配置。

### 0x02 打包配置

如果使用的是 SpringBoot，设置 `includeSystemScope` 为 true。

```
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <configuration>
        <includeSystemScope>true</includeSystemScope>
    </configuration>
</plugin>
```

如果不是 SpringBoot 项目，则设置 `extdirs` 为依赖文件夹。

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <compilerArguments>
            <extdirs>${project.basedir}/src/main/resources/libraries</extdirs>
        </compilerArguments>
    </configuration>
</plugin>
```

