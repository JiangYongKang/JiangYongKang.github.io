---
title: SpringBoot（二）：定时任务
date: 2020-11-11 19:55:28
tags: SpringBoot
categories: SpringBoot 系列
---

在开发过程中，很多需求都需要定时任务才能完成。SpringBoot 提供了简单快捷的实现方式，我们只需要添加一些注解就可以使用。常见的一些需要使用到定时任务的场景。例如：数据备份，消息推送，生成报表等。

<!-- more -->

### 启用定时任务

通过在启动类上添加 `@EnableScheduling` 注解即可启动定时任务。

```java
@Slf4j
@EnableScheduling
@Configuration
public class ScheduleConfig {
}
```

### 设置定时任务

```java
@Slf4j
@Component
public class ScheduledTasks {

    @Scheduled(fixedRate = 1000)
    public void fixedRateTask() {
        log.info("固定速率执行，每秒执行一次");
    }

}
```

执行日志：

```
2023-03-13T21:59:47.739+08:00  INFO 16964 --- [pool-1-thread-1] spring.boot.schedule.schdule.ScheduledTasks      : 固定速率执行，每秒执行一次
2023-03-13T21:59:48.736+08:00  INFO 16964 --- [pool-1-thread-1] spring.boot.schedule.schdule.ScheduledTasks      : 固定速率执行，每秒执行一次
2023-03-13T21:59:49.734+08:00  INFO 16964 --- [pool-1-thread-1] spring.boot.schedule.schdule.ScheduledTasks      : 固定速率执行，每秒执行一次
2023-03-13T21:59:50.730+08:00  INFO 16964 --- [pool-1-thread-1] spring.boot.schedule.schdule.ScheduledTasks      : 固定速率执行，每秒执行一次
```

`@Scheduled` 可以设置两种类型的定时任务，一种是基于固定值间隔的定时任务，另一种是基于 `cron` 表达式的定时任务。`cron` 直接传表达式就可以了。例如：

- `@Scheduled(cron = "*/5 * * * * ?")` 每秒钟执行一次
- `@Scheduled(fixedRate = 1000)` 上一次开始执行时间点之后一秒再执行
- `@Scheduled(fixedDelay = 1000)` 上一次执行完毕时间点之后一秒再执行
- `@Scheduled(initialDelay = 1000, fixedRate = 1000)` ：第一次延迟一秒后执行，之后按 `fixedRate` 的规则每秒执行一次

### 配置定时任务线程池

默认情况下定时任务是单线程的，因为有时候需要执行的定时任务会很多，如果是串行执行会带来一些问题。比如一个很耗时的任务阻塞住了，一些需要短周期循环执行的任务也会卡住。所以可以配置一个线程池来并行执行定时任务。SpringBoot 中可以通过实现 `SchedulingConfigurer` 接口来进行配置。

```Java
@Slf4j
@EnableScheduling
@Configuration
public class ScheduleConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        // 创建一个线程池
        ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
        // 配置核心线程数量 8 个
        taskScheduler.setPoolSize(8);
        // 设置线程名的前缀
        taskScheduler.setThreadNamePrefix("run-schedule-");
        // 设置是否将取消后的任务从队列中清除
        taskScheduler.setRemoveOnCancelPolicy(true);
        // 初始化线程池
        taskScheduler.initialize();
        registrar.setTaskScheduler(taskScheduler);
    }

}
```

然后添加多个不同频率的定时任务，让它们并发执行

```Java
@Slf4j
@Component
public class ScheduledTasks {

    @Scheduled(fixedRate = 1000)
    public void asyncFixedRateTask() {
        log.info("固定速率执行，每秒执行一次");
    }

    @Scheduled(fixedDelay = 1000)
    public void fixedDelayTask() {
        log.info("固定延迟执行，距离上一次调用成功后一秒执行");
    }

    @Scheduled(cron = "*/1 * * * * ?")
    public void cronTask() {
        log.info("使用 Cron 表达式，每秒执行一次");
    }

}
```

根据下面的执行日志，发现线程名已经由 `pool-1-thread-1` 变成了 `run-schedule-*`，定时任务也是在并发的执行。

```
2023-03-13T22:07:34.046+08:00  INFO 1316 --- [ run-schedule-1] spring.boot.schedule.schdule.ScheduledTasks      : 固定速率执行，每秒执行一次
2023-03-13T22:07:34.046+08:00  INFO 1316 --- [ run-schedule-4] spring.boot.schedule.schdule.ScheduledTasks      : 固定延迟执行，距离上一次调用成功后一秒执行
2023-03-13T22:07:35.013+08:00  INFO 1316 --- [ run-schedule-2] spring.boot.schedule.schdule.ScheduledTasks      : 使用 Cron 表达式，每秒执行一次
2023-03-13T22:07:35.044+08:00  INFO 1316 --- [ run-schedule-5] spring.boot.schedule.schdule.ScheduledTasks      : 固定速率执行，每秒执行一次
2023-03-13T22:07:35.059+08:00  INFO 1316 --- [ run-schedule-3] spring.boot.schedule.schdule.ScheduledTasks      : 固定延迟执行，距离上一次调用成功后一秒执行
2023-03-13T22:07:36.002+08:00  INFO 1316 --- [ run-schedule-6] spring.boot.schedule.schdule.ScheduledTasks      : 使用 Cron 表达式，每秒执行一次
2023-03-13T22:07:36.033+08:00  INFO 1316 --- [ run-schedule-1] spring.boot.schedule.schdule.ScheduledTasks      : 固定速率执行，每秒执行一次
2023-03-13T22:07:36.065+08:00  INFO 1316 --- [ run-schedule-4] spring.boot.schedule.schdule.ScheduledTasks      : 固定延迟执行，距离上一次调用成功后一秒执行
2023-03-13T22:07:37.000+08:00  INFO 1316 --- [ run-schedule-7] spring.boot.schedule.schdule.ScheduledTasks      : 使用 Cron 表达式，每秒执行一次
```
