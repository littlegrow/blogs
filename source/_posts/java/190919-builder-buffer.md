---
title: StringBuffer、StringBuilder写入性能及线程安全总结
date: 2019-9-19 19:00:00
description: "StringBuffer线程安全，StringBuilder线程不安全，这个结论都是知道的。StringBuffer为什么线程不安全？既然线程不安全为什么还有存在必要，而不全使用StringBuffer?"
categories:
- StringBuffer
- StringBuilder
---

### `StringBuffer`、`StringBuilder`必要性

<br>

JAVA 中对`String`的定义如下代码片段：

```
public final class String implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    ......

}
```

从定义看到，`String` 是不可变的。当多个字符串进行拼接操作时，将产生很多无用的中间变量，这些中间变量不仅浪费存储空间，也会加重垃圾回收的负担，而字符串拼接场景在日常开发中是极其常见的。因此需要有可变的字符类型来处理这部分操作，这就是`StringBuffer`、`StringBuilder`存在的必要性，均继承至`AbstractStringBuilder`，继承结构如下图：

![](https://haitao.nos.netease.com/20190919113200_8becf919-9780-425b-8c8c-a777f7cd0f64.jpg)

`StringBuilder`线程不安全，`StringBuffer`线程安全，这个结论都是知道的，为什么要这么设计呢？

<br>

### 编码测试写入性能及线程安全性

<br>

为了方便统计耗时，先实现一个时间计算工具类，如下:

```
public class TimeDelta {

    private long time;

    public TimeDelta() {
        this.time = System.currentTimeMillis();
    }

    /**
     * 获取一段时间间隔
     */
    public long getDelta() {
        return System.currentTimeMillis() - time;
    }

    public void renew() {
        time = System.currentTimeMillis();
    }
}
```

写数据使用100个线程，每个线程写入10000个字符，测试代码如下：

```
public class StringBufferBuilderTest {
    /**
     * 测试线程个数
     */
    static final int THREAD_COUNT = 100;

    /**
     * 记录执行耗时
     */
    static TimeDelta timeDelta = new TimeDelta();

    /**
     * 测试 StringBuffer 写入性能及线程安全
     *
     * @throws InterruptedException
     */
    static void testStringBuffer() throws InterruptedException {
        timeDelta.renew();
        CountDownLatch countDownLatch = new CountDownLatch(THREAD_COUNT);
        StringBuffer buffer = new StringBuffer();
        for (int i = 0; i < THREAD_COUNT; i++) {
            new Thread(() -> {
                try {
                    for (int j = 0; j < THREAD_COUNT * 100; j++) {
                        buffer.append('a');
                    }
                } finally {
                    countDownLatch.countDown();
                }
            }).start();
        }
        countDownLatch.await();
        System.out.println("StringBuffer: 耗时 " + timeDelta.getDelta() + "ms");
        System.out.println("StringBuffer: 大小 " + buffer.length() + '\n');
    }

    /**
     * 测试 StringBuilder 写入性能及线程安全
     *
     * @throws InterruptedException
     */
    static void testStringBuilder() throws InterruptedException {
        timeDelta.renew();
        CountDownLatch countDownLatch = new CountDownLatch(THREAD_COUNT);
        StringBuilder builder = new StringBuilder();
        for (int i = 0; i < THREAD_COUNT; i++) {
            new Thread(() -> {
                try {
                    for (int j = 0; j < THREAD_COUNT * 100; j++) {
                        builder.append('a');
                    }
                } finally {
                    countDownLatch.countDown();
                }
            }).start();
        }
        countDownLatch.await();
        System.out.println("StringBuilder: 耗时 " + timeDelta.getDelta() + "ms");
        System.out.println("StringBuilder: 大小 " + builder.length() + '\n');
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println(String.format("测试StringBuffer、StringBuilder性能及线程安全\n写数据使用%s个线程，" +
                "每个线程写入%s个字符\n", THREAD_COUNT, THREAD_COUNT * 100));
        testStringBuffer();
        testStringBuilder();
    }
}
```

<font color="red">注：使用`CountDownLatch`保证每个子线程写数据完毕后主线程继续执行打印缓存尺寸，保证输出准确。</font>

测试结果(不唯一)：

```
测试StringBuffer、StringBuilder性能及线程安全
写数据使用100个线程，每个线程写入10000个字符

StringBuffer: 耗时 177ms
StringBuffer: 大小 1000000

Exception in thread "Thread-102" java.lang.ArrayIndexOutOfBoundsException: 4606
    at java.lang.AbstractStringBuilder.append(AbstractStringBuilder.java:650)
    at java.lang.StringBuilder.append(StringBuilder.java:202)
    at StringBuilderDemo.lambda$testStringBuilder$1(StringBuilderDemo.java:54)
    at java.lang.Thread.run(Thread.java:748)
StringBuilder: 耗时 37ms
StringBuilder: 大小 874110
```

```
测试StringBuffer、StringBuilder性能及线程安全
写数据使用100个线程，每个线程写入10000个字符

StringBuffer: 耗时 221ms
StringBuffer: 大小 1000000

StringBuilder: 耗时 34ms
StringBuilder: 大小 965353
```

<br>

### 测试结果分析

<br>

#### 写入性能

<br>

经多次执行测试代码，`StringBuilder`写入性能均是`StringBuffer`的5倍以上。

写入数据调用`StringBuffer`的方法是：

```
    @Override
    public synchronized StringBuffer append(char c) {
        toStringCache = null;
        super.append(c);
        return this;
    }
```

写入数据调用`StringBuilder`的方法是：

```
    @Override
    public StringBuilder append(char c) {
        super.append(c);
        return this;
    }
```

从以上代码片段看出，`StringBuffer` 的 `append()` 方法均是用 `synchronized` 修饰的，使用同步锁保证线程安全，在多线程业务场景下存在锁竞争，所以性能不如`StringBuilder`。

<br>

#### 线程安全

<br>

`StringBuilder` 的大小总是小于`1000000`，因此部分数据丢失。

这个就很好理解了，`StringBuffer`使用同步锁保证线程安全，可为什么`StringBuilder`会丢失数据呢?

我们看下 `super.append()`：

``` 
    @Override
    public AbstractStringBuilder append(char c) {
        ensureCapacityInternal(count + 1);
        value[count++] = c;
        return this;
    }
```

`count++` 并不是原子操作，因此`StringBuilder`在多线程业务场景下可能存在数据覆盖写入的问题。

<br>

#### 稳定性

<br>

`StringBuilder` 在多线程情景下可能会产生 `ArrayIndexOutOfBoundsException` 异常。

由上一步分析`StringBuilder`多线程下会丢失数据，相比更为致命的是多线程情景下还会偶发异常，这是绝对不能接受的。为什么会有异常？

我们看下`ensureCapacityInternal()`，当存储空间不够时用来扩容

```
    private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        if (minimumCapacity - value.length > 0) {
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));
        }
    }

    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int newCapacity = (value.length << 1) + 2;
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }
```

每次扩容后大小 `length * 2 + 2`，扩容非原子操作，且没有同步代码保护，因此扩容过程中其他线程并发写入可能会引发数组越界异常。

<br>

### 总结

<br>

1. 从源码层面上清楚了`StringBuilder`线程不安全，`StringBuffer`线程安全
2. 单线程业务场景下尽可能使用`StringBuilder`以提高性能
3. 多线程业务场景下使用`StringBuilder`会带来数据丢失和数组越界问题，只能使用`StringBuffer`

<br><br><br><br>
