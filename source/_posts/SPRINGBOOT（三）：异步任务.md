---
title: SpringBoot（三）：异步任务
date: 2020-11-12 19:55:28
tags: SpringBoot
categories: SpringBoot 系列
---

异步任务 Async 和同步任务 Sync 是一个相对的概念。异步任务的执行，不会阻塞当前线程，而是从另一个线程来处理任务。当任务处理完成之后再回调通知主线程。实际的开发过程中有很多与主流程无关或比较耗时的操作都可以通过异步任务来实现，比如：用户注册时发送邮件，数据的批量更新，通知内容的下发。

<!-- more -->

### 启动异步任务

在 SpringBoot 中，通过 `@EnableAsync` 注解来开启异步任务的支持。

```Java
@EnableAsync
@Configuration
public class AsyncConfig {
}
```

### 创建异步任务

在 SpringBoot 中，通过 `@Async` 注解来将一个方法变成异步执行的。需要注意的是，被注解的方法必须是一个 `public` 的非静态方法。

```Java
@Slf4j
@Component
public class AsyncTasks {
    @Async
    public void taskOne() {
        log.info("执行异步任务，无返回值");
    }
}
```

运行单元测试，调用刚才创建的异步任务

```Java
@SpringBootTest
public class AsyncTaskTests {

    @Resource
    private AsyncTask asyncTask;

    @Test
    public void taskOneTest() {
        asyncTask.taskOne();
    }

}
```

根据测试用例的日志可以发现，执行异步任务的线程并不是主线程，而是单独切出来了一个线程在运行。

```
2023-03-14T20:51:43.033+08:00  INFO 11556 --- [           main] spring.boot.async.tests.AsyncTaskTests   : Starting AsyncTaskTests using Java 17.0.6 with PID 11556 (started by vincent in E:\spring-boot-examples\spring-boot-async)
2023-03-14T20:51:43.034+08:00  INFO 11556 --- [           main] spring.boot.async.tests.AsyncTaskTests   : No active profile set, falling back to 1 default profile: "default"
2023-03-14T20:51:43.872+08:00  INFO 11556 --- [           main] spring.boot.async.tests.AsyncTaskTests   : Started AsyncTaskTests in 1.001 seconds (process running for 1.734)
2023-03-14T20:51:44.162+08:00  INFO 11556 --- [         task-1] spring.boot.async.task.AsyncTask         : 执行异步任务，无返回值
```

### 配置线程池

SpringBoot 对于异步任务使用的是默认的 `SimpleAsyncTaskExecutor` 线程池，这个线程池不会复用线程，每次调用都会创建一个新的线程。在任务比较多的情况下，这种频繁创建线程的操作是非常消耗资源的。所以 SpringBoot 提供了一个 `AsyncConfigurer` 接口可以用来实现自定义的线程池。还可以对异步任务执行过程中发生的异常进行捕获处理。

```Java
@Slf4j
@EnableAsync
@Configuration
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor defaultExecutor = new ThreadPoolTaskExecutor();
        defaultExecutor.setMaxPoolSize(32);
        defaultExecutor.setCorePoolSize(16);
        defaultExecutor.setQueueCapacity(16);
        defaultExecutor.setThreadNamePrefix("async-task-");
        defaultExecutor.initialize();
        return defaultExecutor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (exception, method, args) -> {
            log.error("异步任务 {} 执行异常，参数：{}", method.getDeclaringClass().getName() + "#" + method.getName(), args, exception);
        };
    }

}
```

配置好上面的代码之后，重新运行单元测试。可以从日志中观察到，线程的名称已经由原来默认的 `task-1` 变成了自定义的 `async-task-1`。

```
2023-03-14T21:00:59.050+08:00  INFO 21076 --- [           main] spring.boot.async.tests.AsyncTaskTests   : Starting AsyncTaskTests using Java 17.0.6 with PID 21076 (started by vincent in E:\spring-boot-examples\spring-boot-async)
2023-03-14T21:00:59.051+08:00  INFO 21076 --- [           main] spring.boot.async.tests.AsyncTaskTests   : No active profile set, falling back to 1 default profile: "default"
2023-03-14T21:00:59.947+08:00  INFO 21076 --- [           main] spring.boot.async.tests.AsyncTaskTests   : Started AsyncTaskTests in 1.062 seconds (process running for 1.775)
2023-03-14T21:01:00.269+08:00  INFO 21076 --- [   async-task-1] spring.boot.async.task.AsyncTask         : 执行异步任务，无返回值
```

### 异步回调

有些业务场景下，需要多个异步任务执行完成之后再执行最终的处理。这种场景可以在异步任务中返回 `CompletableFuture` 对象来处理。例如：

```Java
@Slf4j
@Component
public class AsyncTask {

    @Async
    public CompletableFuture<String> taskTwo() {
        log.info("执行异步任务，有返回值");
        return CompletableFuture.completedFuture(String.valueOf(System.currentTimeMillis()));
    }

    @Async
    public CompletableFuture<String> taskThree() {
        log.info("执行异步任务，有返回值");
        return CompletableFuture.completedFuture(String.valueOf(System.currentTimeMillis()));
    }

}
```

添加一个新的单元测试，通过 `CompletableFuture.allOf().join()` 方法让两个异步任务都执行完成之后再执行后面的代码。

```Java
@Slf4j
@SpringBootTest
public class AsyncTaskTests {

    @Resource
    private AsyncTask asyncTask;

    @Test
    public void taskTwoTest() {
        CompletableFuture<String> taskTwo = asyncTask.taskTwo();
        CompletableFuture<String> taskThree = asyncTask.taskThree();
        CompletableFuture.allOf(taskTwo, taskThree).join();
        log.info("异步任务执行完成");
    }

}
```

运行上面的单元测试，通过日志可以观察到，程序先运行了 `taskTwo` `taskThree`，这两个任务都完成之后由主线程完成了最后的打印。

```
2023-03-14T21:06:00.246+08:00  INFO 11956 --- [           main] spring.boot.async.tests.AsyncTaskTests   : Starting AsyncTaskTests using Java 17.0.6 with PID 11956 (started by vincent in E:\spring-boot-examples\spring-boot-async)
2023-03-14T21:06:00.247+08:00  INFO 11956 --- [           main] spring.boot.async.tests.AsyncTaskTests   : No active profile set, falling back to 1 default profile: "default"
2023-03-14T21:06:01.139+08:00  INFO 11956 --- [           main] spring.boot.async.tests.AsyncTaskTests   : Started AsyncTaskTests in 1.071 seconds (process running for 1.819)
2023-03-14T21:06:01.495+08:00  INFO 11956 --- [   async-task-2] spring.boot.async.task.AsyncTask         : 执行异步任务，有返回值
2023-03-14T21:06:01.496+08:00  INFO 11956 --- [   async-task-3] spring.boot.async.task.AsyncTask         : 执行异步任务，有返回值
2023-03-14T21:06:01.496+08:00  INFO 11956 --- [           main] spring.boot.async.tests.AsyncTaskTests   : 异步任务执行完成
```