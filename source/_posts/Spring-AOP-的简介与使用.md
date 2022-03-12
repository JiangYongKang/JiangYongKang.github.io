---
title: Spring AOP 的简介与使用
date: 2018-11-19 15:42:13
tags: SpringBoot
categories: 服务端开发
---

### 什么是面向切面编程
AOP 是一种编程范式，并非是 Spring 的独创，它与语言无关，是一种编程思想。AOP 可以帮助我们将通用的逻辑从业务逻辑中分离出来。从项目角度来讲，有一些逻辑散布于程序的各个地方，但它们的逻辑又是千篇一律的相同。比如：我们需要打印每个请求的 URL、请求参数、IP地址，最简单的做法是在每一个 Controller 中去打印出这些信息。如果使用 AOP，我们能够更加轻松的处理这些问题。打印日志并不是使用 AOP 的唯一场景，我们还可以通过切面去统一处理 **声明式事务、缓存、应用安全** 等等。

<!-- more -->

### AOP 相关术语
AOP 作为一种编程范式，已经衍生出了术语它的一些相关术语，例如：**切面、通知、连接点、织入** 等等。为了更好的理解如何在 Spring 中使用 AOP，我们必须对这些术语有一定的认知。

#### 通知(Advice)
Spring AOP 支持五种类型的通知,它们分别定义了切面在什么时候使用，以及定义了切面需要做些什么。
- @Before 前置通知，目标方法被调用之前执行
- @After 后置通知，目标方法完成之后执行
- @AfterReturning 返回通知，目标方法执行成功（未抛出异常）之后执行
- @AfterThrowing 异常通知，目标方法执行失败（抛出异常）之后执行
- @Around 环绕通知，目标方法执行前后都会调用

#### 连接点（JoinPoint）
程序执行的某个特定位置：如类开始初始化前、类初始化后、类某个方法调用前、调用后、方法抛出异常后。一个类或一段程序代码拥有一些具有边界性质的特定点，这些点中的特定点就称为 "连接点"。Spring 仅支持方法的连接点，即仅能在方法调用前、方法调用后、方法抛出异常时以及方法调用前后这些程序执行点织入增强。连接点由两个信息确定：第一是用方法表示的程序执行点，第二是用相对点表示的方位。

#### 切点（Poincut）
每个程序类都拥有多个连接点，如一个拥有两个方法的类，这两个方法都是连接点，即连接点是程序类中客观存在的事物。AOP 通过 "切点" 定位特定的连接点。连接点相当于数据库中的记录，而切点相当于查询条件。切点和连接点不是一对一的关系，一个切点可以匹配多个连接点。在 Spring 中，切点通过
`org.springframework.aop.Pointcut` 接口进行描述，它使用类和方法作为连接点的查询条件，Spring AOP 的规则解析引擎负责切点所设定的查询条件，找到对应的连接点。其实确切地说，不能称之为查询连接点，因为连接点是方法执行前、执行后等包括方位信息的具体程序执行点，而切点只定位到某个方法上，所以如果希望定位到具体连接点上，还需要提供方位信息。

#### 切面（Aspect）
切面由切点和增强（引介）组成，它既包括了横切逻辑的定义，也包括了连接点的定义，Spring AOP 就是负责实施切面的框架，它将切面所定义的横切逻辑织入到切面所指定的连接点中。

#### 引入（Introduction）
引入使我们具备了为类添加一些属性和方法的能力。这样，即使一个业务类原本没有实现某个接口，通过 AOP 的引介功能，我们可以动态地为该业务类添加接口的实现逻辑，让业务类成为这个接口的实现类。

#### 织入（Weaving）
织入是将增强添加对目标类具体连接点上的过程。AOP 像一台织布机，将目标类、增强或引介通过 AOP 这台织布机天衣无缝地编织到一起。根据不同的实现技术，AOP 有三种织入的方式：1. 编译期织入，这要求使用特殊的Java编译器。2. 类装载期织入，这要求使用特殊的类装载器。3. 动态代理织入，在运行期为目标类添加增强生成子类的方式。Spring 采用动态代理织入，而 AspectJ 采用编译期织入和类装载期织入。

#### 代理（Proxy）
一个类被 AOP 织入增强后，就产出了一个结果类，它是融合了原类和增强逻辑的代理类。根据不同的代理方式，代理类既可能是和原类具有相同接口的类，也可能就是原类的子类，所以我们可以采用调用原类相同的方式调用代理类。

### AspectJ 切点表达式
Spring 只支持方法级别的连接点，切点表达式可以通过 `execution` 来定义。比如：`@Pointcut(value = "execution(* com.vincent.springframework.example.Controller.*(..))")` 表示匹配 `com.vincent.springframework.example.Controller` 这个类下面全部的方法。
关于 AspectJ 的 execution 表达式的详细信息，可以查看这篇文章 [Spring AspectJ 的 execution表达式](https://www.cnblogs.com/powerwu/articles/5177662.html)

### 添加 AsepctJ 的依赖
```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.9.1</version>
</dependency>
```

### 使用 XML 定义切面
在配置文件中创建切面，需要通过 Spring 的 **aop** 命名空间来开启。
```xml
<!-- 开启 AOP 的自动代理 -->
<aop:aspectj-autoproxy/>
<!-- 配置包扫描路径 -->
<content:component-scan base-package="com.vincent.spring.framework.example"/>
<!-- 被切入的类 -->
<bean id="messageService" class="com.vincent.spring.framework.example.MessageService"/>
<!-- 切面类 -->
<bean id="messageAspect" class="com.vincent.spring.framework.example.MessageAspect"/>
<!-- 定义切入点已经执行方法 -->
<aop:config>
    <aop:aspect id="messageAspect" ref="messageAspect">
        <aop:before method="beforeAction" pointcut="execution(* com.vincent.spring.framework.example.MessageService.call(..))"/>
        <aop:before method="afterAction" pointcut="execution(* com.vincent.spring.framework.example.MessageService.call(..))"/>
    </aop:aspect>
</aop:config>
```

### 使用 Java 配置定义切面
```java
@Configuration
@EnableAspectJAutoProxy
@ComponentScan(basePackage = "com.vincent.spring.framework.example")
public class AspectJConfig {
}
```
```java
@Aspect
public class MessageAspect {
    // 目标方法执行之前
    @Pointcut("execution(* com.vincent.spring.framework.example.MessageService.call(..))")
    public void beforeAction() {
        System.out.println("before action");
    }
    // 目标方法执行之后
    @Pointcut("execution(* com.vincent.spring.framework.example.MessageService.call(..))")
    public void afterAction() {
        System.out.println("after action");
    }
}
```
