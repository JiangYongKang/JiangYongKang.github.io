---
title: SpringBoot（二）：定时任务
date: 2020-11-11 19:55:28
tags: SpringBoot
categories: SpringBoot 系列
---

在开发过程中，很多需求都需要定时任务才能完成。SpringBoot 提供了简单快捷的实现方式，我们只需要添加一些注解就可以使用。那么什么场景下需要使用到定时任务呢？例如：整点发放优惠券，每天自动更新收益，定时生成统计报表等等。
<!-- more -->

### 启用定时任务

通过在启动类上添加 `@EnableScheduling` 注解即可启动定时任务。

```java
@EnableScheduling
@SpringBootApplication
public class SpringBootScheduling {
    public static void main(String[] args) {
        SpringApplication.run(SpringBootScheduling.class, args);
    }
}
```

### 设置定时任务

```java
@Component
public class ScheduledTasks {

    private static final Logger logger = LoggerFactory.getLogger(ScheduledTasks.class);

    @Scheduled(fixedRate = 1000)
    public void taskOne() {
        logger.info("--> task one");
    }

    @Scheduled(fixedDelay = 1000)
    public void taskTwo() {
        logger.info("--> task two");
    }

    @Scheduled(initialDelay = 1000, fixedDelay = 1000)
    public void taskThree() {
        logger.info("--> task three");
    }

    @Scheduled(cron = "* * * * * *")
    public void taskFour() {
        logger.info("--> task four");
    }
}
```

`@Scheduled` 可以设置两种类型的定时任务，一种是 `fixedRate` 的固定值间隔的定时任务，另一种是基于 `cron` 表达式的定时任务。`cron` 直接传表达式就可以了。固定值分以下几种：

- `@Scheduled(fixedRate = 1000)` 上一次开始执行时间点之后一秒再执行
- `@Scheduled(fixedDelay = 1000)` 上一次执行完毕时间点之后一秒再执行
- `@Scheduled(initialDelay = 1000, fixedRate = 1000)` ：第一次延迟一秒后执行，之后按 `fixedRate` 的规则每秒执行一次