---
title: JAVA锁
date: 2019-9-2
description: "本文总结Java常见锁及使用场景，同时动手编写实例，加深知识的理解，便于今后在适当的业务场景快速使用。"
categories:
- Lock
---

### 锁接口

<br>

Lock 不是Java语言内置的功能，它是一个接口，通过这个接口及具体实现控制临界资源的同步访问。

```
public interface Lock {

    /**
     * 如果锁定不可用，当前线程将被阻塞以进行线程调度，并且在获取锁之前处于休眠状态。
     */
    void lock();

    /**
     * 马上返回，获取lock返回true，不然返回false
     */
    boolean tryLock();

    /**
     * 如果未获得锁，等待一段时间后返回，获取lock返回true，不然返回false
     */
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    /**
     * 如果当前线程未被中断，则获取锁
     * 如果该锁没有被另一个线程保持，则获取该锁并立即返回，将锁的保持计数设置为 1。 
     * 如果当前线程已经保持此锁，则将保持计数加 1，并且该方法立即返回
     * 如果锁被另一个线程保持，则出于线程调度目的，阻塞当前线程，直至获得锁或者线程被其他线程中断
     * 优先考虑响应中断，而不是响应锁的普通获取或重入获取
     */
    void lockInterruptibly() throws InterruptedException;

    /**
     * 释放锁
     */
    void unlock();

    /**
     * 实现等待/通知，提供比wait/notify更丰富的功能
     */
    Condition newCondition();
}

```

<br>

### ReentrantLock 可重入锁

<br>

一个线程在持有这种类型锁的时候，可以再次（多次）申请该锁。

#### 可重入性测试

<br>

```
public class ReentrantLockReentrantTest {
    static class Test {
        private ReentrantLock reentrantLock = new ReentrantLock();

        private void methodA() {
            reentrantLock.lock();
            System.out.println("in methodA");
            methodB();
            reentrantLock.unlock();
            System.out.println("methodA release lock");
        }

        private void methodB() {
            reentrantLock.lock();
            System.out.println("in methodA");
            reentrantLock.unlock();
            System.out.println("methodB release lock");
        }

        public void test() {
            methodA();
        }
    }

    public static void main(String[] args) {
        new Test().test();
    }
}
```

输出结果：

```
in methodA
in methodA
methodB release lock
methodA release lock
```

由结果可看出在 methodA() 未释放锁的情况下 methodB() 依然可以获取该锁。

<br>

#### 公平锁与非公平锁

<br>

ReentrantLock 默认初始化为非公平锁

```
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

获取锁是按照请求的顺序得到的，那么就是公平锁，否则就是非公平锁。

公平锁保证了锁的获取按照FIFO原则，而代价则是进行大量的线程切换。非公平锁可能导致线程饥饿，但却减少了线程切换，提升系统吞吐量。

#### Condition 类

<br>
    
    Condition是个接口，基本的方法就是await()和signal()方法
    Condition依赖于Lock接口，生成一个Condition的基本代码是lock.newCondition() 
    调用Condition的await()和signal()方法，都必须在lock保护之内，就是说必须在lock.lock()和lock.unlock之间才可以使用

借助Condition对象，RetrantLock可以实现类似于Object的wait和notify/notifyAll功能。使用它具有更好的灵活性。
在一个Lock对象里面可以创建多个Condition(对象监视器)实例，线程对象可以注册在指定的Condition对象中，从而可以有选择性的进行线程通知，实现多路通知功能，在调度线程上更加灵活

例：使用`Condition`实现生产者消费者

```
public class ConditionTest {
    private ReentrantLock lock = new ReentrantLock();
    private Condition emptyCondition = lock.newCondition();
    private Condition fullCondition = lock.newCondition();
    private static final int poolSize = 3;
    private static List<Object> pool = new LinkedList<>();

    private void consume() {
        lock.lock();
        try {
            while (pool.isEmpty()) {
                System.out.println(Thread.currentThread().getName() + " pool is empty, wait for produce");
                emptyCondition.await();
            }
            pool.remove(0);
            System.out.println(Thread.currentThread().getName() + " consume one object");
            fullCondition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    private void produce() {
        lock.lock();
        try {
            while (pool.size() >= poolSize) {
                System.out.println(Thread.currentThread().getName() + " pool is full, wait for consume");
                fullCondition.await();
            }
            pool.add(new Object());
            System.out.println(Thread.currentThread().getName() + " produce one object");
            emptyCondition.signalAll();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        ConditionTest conditionTest = new ConditionTest();
        Thread consumer;
        for (int i = 0; i < 3; i++) {
            consumer = new Thread(() -> {
                for (int j = 0; j < 3; j++) {
                    conditionTest.consume();
                }
            }, "consumer-" + i);
            consumer.join();
            consumer.start();
        }
        Thread producer;
        for (int i = 0; i < 3; i++) {
            producer = new Thread(() -> {
                for (int j = 0; j < 3; j++) {
                    conditionTest.produce();
                }
            }, "producer-" + i);
            producer.join();
            producer.start();
        }
    }
}
```

输出(结果不唯一)：

```
consumer-0 pool is empty, wait for produce
consumer-2 pool is empty, wait for produce
consumer-1 pool is empty, wait for produce
producer-0 produce one object
producer-0 produce one object
producer-0 produce one object
producer-2 pool is full, wait for consume
producer-1 pool is full, wait for consume
consumer-0 consume one object
consumer-0 consume one object
consumer-0 consume one object
consumer-2 pool is empty, wait for produce
consumer-1 pool is empty, wait for produce
producer-2 produce one object
producer-2 produce one object
producer-2 produce one object
producer-1 pool is full, wait for consume
consumer-2 consume one object
consumer-2 consume one object
consumer-2 consume one object
consumer-1 pool is empty, wait for produce
producer-1 produce one object
producer-1 produce one object
producer-1 produce one object
consumer-1 consume one object
consumer-1 consume one object
consumer-1 consume one object
```

<br>

### ReentrantReadWriteLock

<br>

ReentrantReadWriteLock允许多个读线程同时访问，但不允许写线程和读线程、写线程和写线程同时访问。

在实际应用中，大部分情况下对共享数据（如缓存）的访问都是读操作远多于写操作，这时ReentrantReadWriteLock能够提供比排他锁更好的并发性和吞吐量。

使用读写锁模拟1个线程写，5个线程读:

```
public class ReadWriteLockTest {
    static class CacheData {
        private ReentrantReadWriteLock readWriteLock = new ReentrantReadWriteLock();
        private Lock readLock = readWriteLock.readLock();
        private Lock writeLock = readWriteLock.writeLock();

        private Map<String, Object> cacheMap = new HashMap<>();

        <T> T getCache(String key) {
            try {
                readLock.lock();
                return (T) cacheMap.get(key);
            } catch (Exception e) {
                return null;
            } finally {
                readLock.unlock();
            }
        }

        <T> void putCache(String key, T value) {
            try {
                writeLock.lock();
                cacheMap.put(key, value);
            } finally {
                writeLock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        CacheData cacheData = new CacheData();
        Random random = new Random(System.currentTimeMillis());
        int readerCount = 5;
        long startTime = System.currentTimeMillis();
        Thread[] readThread = new Thread[readerCount];
        for (int i = 0; i < readerCount; i++) {
            readThread[i] = new Thread(() -> {
                for (int j = 0; j < readerCount; j++) {
                    String key = "cache_" + random.nextInt(5);
                    System.out.println(Thread.currentThread().getName() + " read " + key + ": " + cacheData.getCache(key));
                    try {
                        Thread.sleep(random.nextInt(1000));
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }, "Reader_" + i);
            readThread[i].start();
        }
        Thread writeThread = new Thread(() -> {
            for (int i = 0; i < readerCount; i++) {
                cacheData.putCache("cache_" + i, i);
            }
        });
        writeThread.start();

        writeThread.join();
        for (int i = 0; i < readerCount; i++) {
            readThread[i].join();
        }
        System.out.println("cost " + (System.currentTimeMillis() - startTime));
    }
}
```

输出结果（不唯一）:

```
Reader_0 read cache_3: null
Reader_1 read cache_1: null
Reader_2 read cache_0: null
Reader_3 read cache_0: null
Reader_4 read cache_0: null
Reader_0 read cache_0: 0
Reader_3 read cache_1: 1
Reader_3 read cache_3: 3
Reader_2 read cache_3: 3
Reader_0 read cache_3: 3
Reader_4 read cache_4: 4
Reader_2 read cache_1: 1
Reader_2 read cache_4: 4
Reader_0 read cache_3: 3
Reader_3 read cache_2: 2
Reader_1 read cache_0: 0
Reader_2 read cache_0: 0
Reader_0 read cache_1: 1
Reader_4 read cache_1: 1
Reader_3 read cache_1: 1
Reader_1 read cache_4: 4
Reader_4 read cache_3: 3
Reader_1 read cache_4: 4
Reader_1 read cache_4: 4
Reader_4 read cache_1: 1
cost 2763
```

<br>

### 总结

<br>

本文总结Java常见锁及使用场景，同时动手编写实例，加深知识的理解，便于今后在适当的业务场景快速使用。

<br><br><br><br>

