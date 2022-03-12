---
title: 本地线程变量 ThreadLocal
date: 2018-01-16 17:05:17
tags: ThreadLocal
categories: Java
---

`ThreadLocal`为每个线程的中并发访问的数据提供一个副本，通过访问副本来运行业务，这样的结果是耗费了内存，单大大减少了线程同步所带来性能消耗，也减少了线程并发控制的复杂度。`ThreadLocal` 不能使用原子类型，只能使用 `Object` 类型。`ThreadLocal` 的使用比 `synchronized` 要简单得多。`ThreadLocal` 和 `Synchonized` 都用于解决多线程并发访问。但是 `ThreadLocal` 与 `synchronized` 有本质的区别。`synchronized` 是利用锁的机制，使变量或代码块在某一时该只能被一个线程访问。而 `ThreadLocal` 为每一个线程都提供了变量的副本，使得每个线程在某一时间访问到的并不是同一个对象，这样就隔离了多个线程对数据的数据共享。而 `synchronized` 却正好相反，它用于在多个线程间通信时能够获得数据共享。`synchronized` 用于线程间的数据共享，而 `ThreadLocal` 则用于线程间的数据隔离。当然 `ThreadLocal` 并不能替代 `synchronized`,它们处理不同的问题域。`synchronized` 用于实现同步机制，比 `ThreadLocal` 更加复杂。

<!-- more -->

```Java
// 本地线程变量一般是 static final 的，withInitial 方法可以设定它的初始值。
private static final ThreadLocal<Integer> local = ThreadLocal.withInitial(() -> 0);
```
```Java
// 定义一个方法，在控制台中打印在修改前后 local 的值，以及当前线程的名称。
private static void setLocal() {
    String threadName = Thread.currentThread().getName();
    System.out.println(threadName + " before set: " + local.get());
    local.set(local.get() + 1);
    System.out.println(threadName + " after set: " + local.get());
}

// 在主方法中创建多个线程，分别调用 setLocal 方法
public static void main(String[] args) {
    new Thread(ThreadLocalTest::setLocal).start();
    new Thread(ThreadLocalTest::setLocal).start();
    new Thread(ThreadLocalTest::setLocal).start();

    System.out.println("main thread: " + local.get());
}
```
```
输出如下：

Thread-0 before set: 0
Thread-0 after set: 1
Thread-1 before set: 0
Thread-1 after set: 1
main thread: 0
Thread-2 before set: 0
Thread-2 after set: 1
```
`ThreadLocal` 会在每个线程第一次访问时都创建一个变量副本，所以虽然看似在操作同一个对象，但其实都是在操作当前线程的克隆出来的副本。每一个线程中都有一个对象的副本，所以它的值才没有被累加。
