---
title: 浅谈 Object 类中的方法
date: 2021-03-27 15:53:14
tags: Java
categories: 服务端开发
---

Object 类是 Java 中所有类的父类，在 Java 中每个类都是由 Object 类扩展来的。所以，Object 中的实例方法也是所有 Java 对象可以使用的。

<!-- more -->

### public final native Class<?> getClass();

该方法用于获得运行时的类型。该方法返回的是此 Object 对象的类对象运行时类对象 Class。获取 Class 对象还可以使用 `Object.class` 或者 `Class.forName()` 进行获取

### public native int hashCode();

该方法返回当前对象的哈希值，这个值是对象的物理地址，取值范围是 `-2^31 ~ 2^31 - 1`，通常和 `equals()` 方法一起进行重写，确保相等的两个对象拥有相等的 `hashCode`。Java 规范对 hashCode 有以下几点约束：

- 对同一对象多次调用 `hashCode()` 方法时，必须一致地返回相同的整数，前提是将对象进行 `equals()` 比较时所用的信息没有被修改
- 如果两个对象 `x.equals(y)` 方法返回 true，则 x、y 这两个对象的 `hashCode()` 必须相等
- 如果两个对象 `x.equals(y)` 方法返回 false，则 x、y 这两个对象的 `hashCode()` 可以相等也可以不等。 但是，为不相等的对象生成不同整数结果可以提高哈希表的性能。
- 默认的 `hashCode()` 是将内存地址转换为的哈希值，重写过后就是自定义的计算方式。也可以通过 `System.identityHashCode(Object o)` 来返回原本的 `hashCode()`。

### public boolean equals(Object o)

该方法用来比较两个对象的内容是否相等，默认情况下是比较引用是否指向同一对象，除非被子类重写了。

```Java
public boolean equals(Object obj) {
    return (this == obj);
}
```

通常在重写 `equals()` 方法时需要遵守以下几条约定：

- 自反性：即 `x.equals(x)` 返回 true，x 不为 null
- 对称性：即 `x.equals(y)` 与 `y.equals(x)` 的结果相同，x 与 y 不为 null
- 传递性：即 `x.equals(y)` 结果为 true, `y.equals(z)` 结果为 true，则 `x.equals(z)` 结果也必须为 true
- 一致性：即 `x.equals(y)` 返回 true 或 false，在未更改 `equals()` 方法使用的参数条件下，多次调用返回的结果也必须一致。x 与 y 不为 null
- 如果 x 不为 null, `x.equals(null)` 返回 false

### protected native Object clone() throws CloneNotSupportedException;

此方法返回当前对象的一个副本。需要注意的是，Object 类中的 `clone()` 方法是浅拷贝。这是一个 `protected` 方法，提供给子类重写。但需要实现 `Cloneable` 接口，这是一个标记接口，如果没有实现，当调用 `Object.clone()` 方法，会抛出 `CloneNotSupportedException` 异常，如下：

```Java
public class CloneTest implements Cloneable {
    private int value;
    @Override
    protected CloneTest clone() throws CloneNotSupportedException {
        return (CloneTest) super.clone();
    }
}
```

### public String toString()

返回该对象的字符串表示，默认的 `toString()` 方法，只是将当前类的全限定性类名 + `@` + 十六进制的 `hashCode()` 值。

### public final void wait() throws InterruptedException

该方法还有两个重载的版本，分别是：

- `public final native void wait(long timeout) throws InterruptedException;`
- `public final void wait(long timeout, int nanos) throws InterruptedException;`

这三个方法是用来线程间通信用的，作用是阻塞当前线程，等待其他线程调用 `notify()` / `notifyAll()` 方法将其唤醒。这三个方法都是被 `final` 修饰过的，不可以被子类重写。

调用 `wait()` 方法的一些注意事项：

1. `wait()` 方法只能在当前线程获取到对象的锁监视器之后才能调用，否则会抛出 `IllegalMonitorStateException` 异常。
2. 调用 `wait()` 方法，线程会将锁监视器进行释放。而 `Thread.sleep()`，`Thread.yield()` 并不会释放锁。
3. `wait()` 方法会一直阻塞，直到其他线程调用当前对象的 `notify()` / `notifyAll()` 方法将其唤醒。而 `wait(long timeout)` 是等待给定超时时间内（单位毫秒），如果还没有调用 `notify()` / `nofiyAll()` 会自动唤醒。`wait(long timeout, int nanos)` 如果第二个参数大于 0 并且小于 999999，则第一个参数 +1 作为超时时间。

### public final native void notify();

随机唤醒之前在当前对象上调用 `wait()` 方法的一个线程。

### public final native void notifyAll();

唤醒所有之前在当前对象上调用 `wait()` 方法的线程。为什么 `wait()` / `notify()` 方法要放到 Object 中呢？因为每个对象都可以成为锁监视器对象，所以放到 Object 中，所有对象就都可以使用了。

### protected void finalize() throws Throwable { }

此方法是在垃圾回收之前，JVM 会调用此方法来清理资源。此方法可能会将对象重新置为可达状态，导致 JVM 无法进行垃圾回收。这个方法需要以下几点：

1. 永远不要主动调用某个对象的 `finalize()` 方法，该方法由垃圾回收机制自己调用
2. `finalize()` 何时被调用，是否被调用具有不确定性
3. 当 JVM 执行可恢复对象的 `finalize()` 可能会将此对象重新变为可达状态
4. 当 JVM 执行 `finalize()` 方法时出现异常，垃圾回收机制不会报告异常，程序继续执行
