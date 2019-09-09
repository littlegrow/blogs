---
title: CountDownLatch、Semaphore、CyclicBarrier
date: 2019-9-3
description: "本文总结CountDownLatch、Semaphore和CyclicBarrier概念及使用举例"
categories:
- 并发
---

### CountDownLatch

<br>

CountDownLatch是一个计数器闭锁，通过它可以完成类似于阻塞当前线程的功能，即：一个线程或多个线程一直等待，直到其他线程执行的操作完成。如我们的API接口内部依赖多个三方外部服务，那串行调用接口必然耗时很久，并行调用可使用此计数器锁控制。

例: 多个日志收集进程都收集日志完毕后再进行后续处理。

```
import java.util.Random;
import java.util.concurrent.CountDownLatch;

public class CountDownLatchTest {
    public static void main(String[] args) {
        int logCollectThreadCount = 3;
        CountDownLatch countDownLatch = new CountDownLatch(logCollectThreadCount);
        Random random = new Random();
        for (int i = 0; i < logCollectThreadCount; i++) {
            new Thread(() -> {
                System.out.println(Thread.currentThread().getName() + " 开始收集日志");
                try {
                    Thread.sleep(random.nextInt(100));
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                countDownLatch.countDown();
                System.out.println(Thread.currentThread().getName() + " 收集日志完毕");
            }, "Collect-thread-" + i).start();
        }
        try {
            countDownLatch.await();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("收集日志完成");
    }
}
```

输出结果(不唯一) ：

```
Collect-thread-0 开始收集日志
Collect-thread-2 开始收集日志
Collect-thread-1 开始收集日志
Collect-thread-2 收集日志完毕
Collect-thread-1 收集日志完毕
Collect-thread-0 收集日志完毕
收集日志完成
```

<br>

### Semaphore

<br>

Semaphore 是 synchronized 的加强版，作用是控制线程的并发数量。

例：控制同时只有2个线程执行。

```
public class SemaphoreTest {
    public static void main(String[] args) {
        Semaphore semaphore = new Semaphore(2);
        for (int i = 0; i < 5; i++) {
            new Thread(() -> {
                try {
                    semaphore.acquire();
                    System.out.println(Thread.currentThread().getName() + " enter");
                    Thread.sleep(100);
                    System.out.println(Thread.currentThread().getName() + " exit");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    semaphore.release();
                }
            }, "Thread-" + i).start();
        }
    }
}
```

输出（不唯一）

```
Thread-0 enter
Thread-1 enter
Thread-1 exit
Thread-3 enter
Thread-0 exit
Thread-2 enter
Thread-2 exit
Thread-3 exit
Thread-4 enter
Thread-4 exit
```

由输出结果可看出，同时只有2个线程在执行

<br>

### CyclicBarrier

<br>

多个线程之间相互等待，只有当每个线程都准备就绪后，才能各自继续往下执行后面的操作。

例：所有工人准备完毕后开始工作

```
public class CyclicBarrierTest {
    static class Worker extends Thread {
        private CyclicBarrier cyclicBarrier;

        Worker(CyclicBarrier cyclicBarrier, String name) {
            super(null, null, name, 0);
            this.cyclicBarrier = cyclicBarrier;
        }

        @Override
        public void run() {
            super.run();
            System.out.println(getName() + " 开始等待其他工人");
            try {
                cyclicBarrier.await();
                System.out.println(getName() + " 开始执行");
                Thread.sleep(1000);
                System.out.println(getName() + " 执行完毕");
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        CyclicBarrier cyclicBarrier = new CyclicBarrier(3);
        for (int i = 0; i < 3; i++) {
            Worker worker = new Worker(cyclicBarrier, "worker-" + i);
            worker.start();
        }
    }
}
```

输出结果（不唯一）

```
worker-2 开始等待其他工人
worker-1 开始等待其他工人
worker-0 开始等待其他工人
worker-1 开始执行
worker-0 开始执行
worker-2 开始执行
worker-0 执行完毕
worker-2 执行完毕
worker-1 执行完毕
```


<br><br><br><br>
