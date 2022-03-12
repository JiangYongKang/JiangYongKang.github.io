---
title: 高性能队列 Disruptor 使用教程
date: 2019-12-14 13:49:17
tags: Disruptor
categories: 服务端开发
---

Disruptor 是英国外汇交易公司LMAX开发的一个高性能队列，研发的初衷是解决内存队列的延迟问题（在性能测试中发现竟然与I/O操作处于同样的数量级）。基于 Disruptor 开发的系统单线程能支撑每秒 600 万订单，2010 年在 QCon 演讲后，获得了业界关注。2011年，企业应用软件专家 Martin Fowler 专门撰写长文介绍。同年它还获得了 Oracle 官方的 Duke 大奖。
<!-- more -->

### Disruptor 是什么
从数据结构上来看，Disruptor 是一个支持 **生产者 -> 消费者** 模式的 **环形队列**。能够在 **无锁** 的条件下进行并行消费，也可以根据消费者之间的依赖关系进行先后消费次序。本文将演示一些经典的场景如何通过 Disruptor 去实现。


### 添加依赖
```xml
<dependency>
    <groupId>com.lmax</groupId>
    <artifactId>disruptor</artifactId>
    <version>3.4.2</version>
</dependency>
```

### 单生产者单消费者模式
首先创建一个 `OrderEvent` 类，这个类将会被放入环形队列中作为消息内容。
```Java
@Data
public class OrderEvent {
    private String id;
}
```
创建 `OrderEventProducer` 类，它将作为生产者使用。
```Java
public class OrderEventProducer {
    private final RingBuffer<OrderEvent> ringBuffer;
    public OrderEventProducer(RingBuffer<OrderEvent> ringBuffer) {
        this.ringBuffer = ringBuffer;
    }
    public void onData(String orderId) {
        long sequence = ringBuffer.next();
        try {
            OrderEvent orderEvent = ringBuffer.get(sequence);
            orderEvent.setId(orderId);
        } finally {
            ringBuffer.publish(sequence);
        }
    }
}
```
创建 `OrderEventHandler` 类，并实现 `EventHandler<T>` 和 `WorkHandler<T>` 接口，作为消费者。
```Java
@Slf4j
public class OrderEventHandler implements EventHandler<OrderEvent>, WorkHandler<OrderEvent> {
    @Override
    public void onEvent(OrderEvent event, long sequence, boolean endOfBatch) {
        log.info("event: {}, sequence: {}, endOfBatch: {}", event, sequence, endOfBatch);
    }
    @Override
    public void onEvent(OrderEvent event) {
        log.info("event: {}", event);
    }
}
```
创建完上面三个类，我们就已经具备了 `事件类`、`生产者`、`消费者` 这三个要素了。接下来我们通过一个主方法来演示这一系列流程。
```Java
@Slf4j
public class DisruptorDemo {
    public static void main(String[] args) throws InterruptedException {
        Disruptor<OrderEvent> disruptor = new Disruptor<>(
                OrderEvent::new,
                1024 * 1024,
                Executors.defaultThreadFactory(),
                ProducerType.SINGLE,
                new YieldingWaitStrategy()
        );
        disruptor.handleEventsWith(new OrderEventHandler());
        disruptor.start();
        RingBuffer<OrderEvent> ringBuffer = disruptor.getRingBuffer();
        OrderEventProducer eventProducer = new OrderEventProducer(ringBuffer);
        eventProducer.onData(UUID.randomUUID().toString());
    }
}
```

### 单生产者多消费者
如果消费者是多个，只需要在调用 `handleEventsWith` 方法时将多个消费者传递进去。下面的代码传递了两个消费者。
```Diff
- disruptor.handleEventsWith(new OrderEventHandler());
+ disruptor.handleEventsWith(new OrderEventHandler(), new OrderEventHandler());
```
上面传入的两个消费者会重复消费每一条消息，如果想实现一条消息在有多个消费者的情况下，只会被一个消费者消费，那么需要调用 `handleEventsWithWorkerPool` 方法。
```Diff
- disruptor.handleEventsWith(new OrderEventHandler());
+ disruptor.handleEventsWithWorkerPool(new OrderEventHandler(), new OrderEventHandler());
```

### 多生产者多消费者
在实际开发中，多个生产者发送消息，多个消费者处理消息才是常态。这一点，Disruptor 也是支持的。多生产者多消费者的代码如下：
```Java
@Slf4j
public class DisruptorDemo {
    public static void main(String[] args) throws InterruptedException {
        Disruptor<OrderEvent> disruptor = new Disruptor<>(
                OrderEvent::new,
                1024 * 1024,
                Executors.defaultThreadFactory(),
                // 这里的枚举修改为多生产者
                ProducerType.MULTI,
                new YieldingWaitStrategy()
        );
        disruptor.handleEventsWithWorkerPool(new OrderEventHandler(), new OrderEventHandler());
        disruptor.start();
        RingBuffer<OrderEvent> ringBuffer = disruptor.getRingBuffer();
        OrderEventProducer eventProducer = new OrderEventProducer(ringBuffer);
        // 创建一个线程池，模拟多个生产者
        ExecutorService fixedThreadPool = Executors.newFixedThreadPool(100);
        for (int i = 0; i < 100; i++) {
            fixedThreadPool.execute(() -> eventProducer.onData(UUID.randomUUID().toString()));
        }
    }
}
```

### 消费者优先级
Disruptor 可以做到的事情远远不止上面的内容。在实际场景中，我们通常会因为业务逻辑而形成一条消费链。比如一个消息必须由 `消费者A -> 消费者B -> 消费者C` 的顺序依次进行消费。在配置消费者时，可以通过 `.then` 方法去实现。如下：
```Java
disruptor.handleEventsWith(new OrderEventHandler())
         .then(new OrderEventHandler())
         .then(new OrderEventHandler());
```
当然，`handleEventsWith` 与 `handleEventsWithWorkerPool` 都是支持 `.then` 的，它们可以结合使用。比如可以按照 `消费者A -> (消费者B 消费者C) -> 消费者D` 的消费顺序
```Java
disruptor.handleEventsWith(new OrderEventHandler())
         .thenHandleEventsWithWorkerPool(new OrderEventHandler(), new OrderEventHandler())
         .then(new OrderEventHandler());
```

### 总结
以上就是 Disruptor 高性能队列的常用方法。其实 `生成者 -> 消费者` 模式是很常见的，通过一些消息队列也可以轻松做到上述的效果。不同的地方在于，Disruptor 是在内存中以队列的方式去实现的，而且是无锁的。这也是 Disruptor 为什么高效的原因。

### 参考链接
- [http://lmax-exchange.github.io/disruptor/](http://lmax-exchange.github.io/disruptor/)
- [https://tech.meituan.com/2016/11/18/disruptor.html](https://tech.meituan.com/2016/11/18/disruptor.html)
- [https://www.cnblogs.com/pku-liuqiang/p/8544700.html](https://www.cnblogs.com/pku-liuqiang/p/8544700.html)
- [http://ifeve.com/disruptor/](http://ifeve.com/disruptor/)
