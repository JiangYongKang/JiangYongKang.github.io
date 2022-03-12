---
title: Spring Bean 装配和依赖注入
date: 2018-11-18 14:41:05
tags: SpringBoot
categories: 服务端开发
---

Spring IOC 容器的依赖注入工作可以分为两个阶段。第一个阶段可以认为是构建和收集 Bean 定义的阶段，在这个阶段中，我们可以通过 xml 或者 Java 代码的方式定义一些 Bean，然后通过手动组装或者让容器基于某种规则自动扫描的形式，将这些 Bean 定义收集到 IOC 容器中。下面是以XML配置的形式来注册一些 Bean

<!-- more -->

```xml
<!-- 声明一个 JavaBean -->
<bean id="foo" class="com.vincent.spring.Foo"/>

<!-- 声明一个 JavaBean 并注入构造参数 -->
<bean id="dataSource" class="com.vincent.spring.DataSource">
    <constructor-arg value="root"/>
    <constructor-arg value="root"/>
</bean>

<!-- 指定参数名称进行装配 -->
<bean id="dataSource" class="com.vincent.spring.DataSource">
    <property name="username" value="root"/>
    <property name="password" value="root"/>
</bean>

<!-- 装配构造函数参数是 List 类型的 JavaBean -->
<bean id="foo" class="com.vincent.spring.Foo">
    <constructor-arg>
        <list>
            <value>100</value>
            <value>200</value>
            <value>300</value>
        </list>
    </constructor-arg>
</bean>

<!-- 装配构造函数参数是 Set 类型的 JavaBean -->
<bean id="foo" class="com.vincent.spring.Foo">
    <constructor-arg>
        <set>
            <value>100</value>
            <value>200</value>
            <value>300</value>
        </set>
    </constructor-arg>
</bean>
```
```java
// 声明一个组件
@Component

// 声明一个组件并设置它的ID
@Component("foo")
```
```java
// 自动装配一个 JavaBean
@Resource

// 装配指定ID的 JavaBean
@Resource("foo")

// 通过构造函数装配 JavaBean
public class Foo {

    private Bar bar;

    @Autowired
    public Foo(Bar bar) {
        this.bar = bar;
    }
}

// 忽略没有找到的 JavaBean，但是这样做容易抛出 NPE
@Resource(required = false)
```

如果觉得逐个收集 Bean 定义麻烦，想批量的将 Bean 收集并注册到容器之中，我们也可以配置批量扫描注册 Bean 的方式进行。下面分别是基于 XML 和 Java 配置的方式进行批量注册。
```xml
<!-- 扫描一个或多个制定包下的组件 -->
<!-- 与 Java 配置不同，使用 XML 配置必须指定扫描包的范围 -->
<context:component-scan base-package="com.vincent.spring.framework.example"/>
```
```java
// 自动扫描当前包下的组件
@ComponentScan

// 扫描一个或多个制定包下的组件
@ComponentScan(basePackages = {
    "com.vincent.spring.framework.example.web", "com.vincent.spring.framework.example.service"
})

// 扫描指定类所在的包
@ComponentScan(basePackageClasses = com.vincent.spring.framework.example.SpringContextConfig.class)
```
当 Bean 都已经成功注册到 IOC 容器中后，IOC 容器会分析这些 Bean 之间的依赖关系，根据他们之间的依赖关系先后组装它们。比如 IOC 容器发现 JdbcCTemplate 这个 Bean 依赖于 dataSource，它就会将这个 Bean 注入给依赖它的那个 Bean 直到所有的 Bean 都依赖注入完成。至此，所有的 Bean 都组装完成。

代码案例：[spring-framework-bean-example](https://github.com/JiangYongKang/spring-framework-examples/tree/master/spring-framework-bean-example)
