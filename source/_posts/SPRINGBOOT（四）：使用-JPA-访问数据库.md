---
title: SpringBoot（四）：使用 JPA 访问数据库
date: 2020-11-13 15:44:53
tags: SpringBoot
categories: SpringBoot 系列
---

[**Spring Data JPA**](https://docs.spring.io/spring-data/data-jpa/docs/current/reference/html) 是一个基于 **Hibernate** 二次封装的 **ORM** 框架，它提供了访问数据库所需要的一些方法，可以与 SpringBoot 快速的集成再一起，打大简化我们的开发。本文将简单介绍如何整合，使用 Spring Data JPA。

<!-- more -->

### 引入依赖

在原有 SpringBoot 项目的基础上，仅需要添加一个 `spring-boot-starter-data-jpa` 依赖项就可以了。相应的数据库驱动可以根据自己的数据库进行替换，本文使用的是 H2。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 配置文件

因为我这里使用的是 H2 数据库，所以仅需要配置数据库连接地址和驱动类。`show-sql` 选项开启后可以在日志中打印 SQL 语句。

```yml
spring:
  datasource:
    url: jdbc:h2:mem:test
    driver-class-name: org.h2.Driver
  jpa:
    show-sql: true
```

### 访问数据库

首选需要创建一个实体类，它的成员属性需要与数据库中的字段一一对应：

```java
@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String firstName;
    private String lastName;

    public User() {}

    public User(Long id, String firstName, String lastName) {
        this.id = id;
        this.firstName = firstName;
        this.lastName = lastName;
    }

    // Getter And Setter ...
}
```

定义实体类对应的 Repository 接口，继承 CrudRepository 接口，CrudRepository 接口提供了访问数据的基本操作，如 `save(), findById(), existsById(), deleteById()` 等。

```java
public interface UserRepository extends CrudRepository<User, Long> {
}
```

好了，现在通过 `UserRepository` 就可以进行数据库的增删改查了，我们来测试一下。定义一个 `UserRunner` 类，实现 `CommandLineRunner` 接口。

```java
@Slf4j
@Component
public class UserRunner implements CommandLineRunner {

    @Resource
    private UserRepository userRepository;

    @Override
    public void run(String... args) {
        // 插入一条数据
        userRepository.save(new User(1L, "Vincent", "Jiang"));
        // 通过 ID 查询数据
        log.info("User: {}", userRepository.findById(1L));
        // 通过 ID 删除数据
        userRepository.deleteById(1L);
        // 查询数据是否还存在
        log.info("User exists: {}", userRepository.existsById(1L));
    }
}
```

CommandLineRunner 接口会在 SpringBoot 项目启动时自动执行它的 run 方法。启动项目应该可以看到以下日志输出：

```text
Hibernate: select user0_.id as id1_0_0_, user0_.first_name as first_na2_0_0_, user0_.last_name as last_nam3_0_0_ from user user0_ where user0_.id=?
Hibernate: call next value for hibernate_sequence
Hibernate: insert into user (first_name, last_name, id) values (?, ?, ?)
Hibernate: select user0_.id as id1_0_0_, user0_.first_name as first_na2_0_0_, user0_.last_name as last_nam3_0_0_ from user user0_ where user0_.id=?
2020-11-13 15:01:25.317  INFO 14148 --- [           main] spring.boot.data.jpa.runner.UserRunner   : User: Optional[User(id=1, firstName=Hello, lastName=World)]
Hibernate: select user0_.id as id1_0_0_, user0_.first_name as first_na2_0_0_, user0_.last_name as last_nam3_0_0_ from user user0_ where user0_.id=?
Hibernate: delete from user where id=?
Hibernate: select count(*) as col_0_0_ from user user0_ where user0_.id=?
2020-11-13 15:01:25.374  INFO 14148 --- [           main] spring.boot.data.jpa.runner.UserRunner   : User exists: false
```

### 查询表达式

真实的业务场景往往是复杂多变的，CrudRepository 无法满足业务需求时，我们可以自定义查询方法进行数据访问。在这里，我们定义一个根据 `firstName` 进行查询的方法。

```java
public interface UserRepository extends CrudRepository<User, Long> {
    List<User> findByFirstName(String firstName);
}
```

当 Spring Data JAP 创建一个新的 Repository 实例时，它将分析接口定义的所有方法，并尝试根据方法名称自动生成查询。上面的 findByFirstName 在被调用时将会生成 `select * from user where first_name = ?` 语句，除了上面的 findBy 之外，更多的关键字可以在 [Query Creation](https://docs.spring.io/spring-data/data-jpa/docs/current/reference/html/#jpa.query-methods.query-creation) 找到。

### 更复杂的查询

虽然通过方法名生成查询语句的方式非常快速简洁，但是这种方法是有局限性的，复杂的查询会导致方法名称又臭又长。所以 Spring Data JPA 提供了 `@Query` 注解，方便开发者对查询语句进行更细粒度的控制。

```java
public interface UserRepository extends CrudRepository<User, Long> {
    @Query("select u from User u where u.lastName = :lastName")
    List<User> findByLastName(@Param("lastName") String firstName);
}
```

上面的 findByLastName 在被调用时将会生成 `select * from user where last_name = ?` 语句
