---
title: 热点缓存简单实现
date: 2019-8-29 21:00:00
description: "短时间内大量访问某些数据可认定为热点数据，如果这部分数据均从缓存或数据库获取可能造成缓存穿透、缓存击穿等问题，将这部分数据在本地做缓存很有必要。"
categories:
- 内存缓存
---

### 背景

<br>

短时间内大量访问某些数据可认定为热点数据，如果这部分数据均从缓存或数据库获取可能造成缓存穿透、缓存击穿等问题，将这部分数据在本地做缓存很有必要。
另外后台开发一些管理资源类单机系统时，需要用到少量数据的缓存服务，申请redis等缓存资源太过重量，可考虑将这部分数据缓存在内存中，防止每次请求都查库，提高单机系统吞吐率。

<br>

知识点积累：

* 缓存穿透

<br>

指查询一个数据库一定不存在的数据，大量的请求同时压向数据库，导致系统瘫痪。这时用户很可能是攻击者，攻击会导致数据库压力过大。

解决方案：

    接口层增加校验，如参数有效性拦截
    从缓存取不到的数据，在数据库中也没有取到，这时也可以将key-value对写为key-null，缓存有效时间可以设置短点（设置太长会导致正常情况也没法使用）。这样可以防止攻击用户反复用同一个id暴力攻击

<br>

* 缓存击穿

<br>

缓存击穿是指缓存中没有但数据库中有的数据（一般是缓存时间到期），这时由于并发用户特别多，同时读缓存没读到数据，又同时去数据库去取数据，引起数据库压力瞬间增大，造成过大压力

    设置热点数据
    访问数据库加互斥锁
    访问数据库前设置热点数据默认值为空，短时间内其他请求返回默认值，查库成功后设置缓存，后续请求正常

本文讲述我如何处理这部分数据。

<br>

### 缓存接口

<br>

以求最简单的方式缓存这部分数据，因此接口设计越简单越好，定义了如下几个接口：

<br>

```
public interface IMemCacheService {
    /**
     * 缓存数据
     *
     * @param key   缓存Key
     * @param value 缓存内容
     * @return 缓存是否添加成功
     */
    <T> boolean saveCache(String key, T value);

    /**
     * 缓存数据
     *
     * @param key       缓存Key
     * @param value     缓存内容
     * @param aliveTime 缓存存活时间
     * @return 缓存是否添加成功
     */
    <T> boolean saveCache(String key, T value, long aliveTime);

    /**
     * 读取缓存
     *
     * @param key 缓存Key
     * @return 缓存内容, 无缓存返回 null
     */
    <T> T getCache(String key);

    /**
     * 清理缓存
     */
    boolean clear();
}
```

<br>

### 缓存实现

<br>

设计要点：

<br>

    软引用持有缓存，在系统资源紧张时可被回收；
    支持多线程业务场景；
    支持设置默认存活时间和自定义存活时间；
    除主动查找判断存活状态以外，需要定时清理失效缓存，防止无限增长。

<br>

```
import org.springframework.stereotype.Service;

import java.lang.ref.SoftReference;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

@Service
public class MemCacheServiceImpl implements IMemCacheService {
    /**
     * 默认存活时间5分钟
     */
    private static final long ALIVE_TIME_DEFAULT = 5 * 60 * 1000;
    /**
     * 定时清理任务执行间隔，默认 20 分钟
     */
    private static final int CLEAR_TIME_DEFAULT = 20;

    /**
     * 使用 {@link ConcurrentHashMap} 存储缓存，支持多线程业务场景
     */
    private ConcurrentHashMap<String, SoftReference<CacheItem>> cacheMap = new ConcurrentHashMap<>();

    /**
     * 单线程池，间隔定时执行缓存清理任务
     */
    private final ScheduledExecutorService executorService = Executors.newSingleThreadScheduledExecutor();
    private final Object lock = new Object();
    private boolean submitCleanRunnable = false;
    /**
     * 缓存清理任务
     */
    private Runnable cleanRunnable = () -> {
        List<String> waitRemoveKeys = new ArrayList<>();
        // 找出失效缓存
        for (Map.Entry<String, SoftReference<CacheItem>> entry : cacheMap.entrySet()) {
            CacheItem item = entry.getValue().get();
            if (item == null || !item.isAlive()) {
                waitRemoveKeys.add(entry.getKey());
            }
        }
        // 清理失效缓存
        for (String key : waitRemoveKeys) {
            cacheMap.remove(key);
        }
    };

    @Override
    public <T> boolean saveCache(String key, T value) {
        return saveCache(key, value, ALIVE_TIME_DEFAULT);
    }

    @Override
    public <T> boolean saveCache(String key, T value, long aliveTime) {
        // 双重校验锁保证多线程业务场景下清理任务仅被提交一次
        if (!submitCleanRunnable) {
            synchronized (lock) {
                if (!submitCleanRunnable) {
                    executorService.scheduleAtFixedRate(cleanRunnable, CLEAR_TIME_DEFAULT, CLEAR_TIME_DEFAULT, TimeUnit.MINUTES);
                }
                submitCleanRunnable = true;
            }
        }
        CacheItem cacheItem = new CacheItem(aliveTime, value);
        cacheMap.put(key, new SoftReference<>(cacheItem));
        return cacheMap.containsKey(key);
    }

    @Override
    public <T> T getCache(String key) {
        if (cacheMap.containsKey(key)) {
            CacheItem item = cacheMap.get(key).get();
            if (item != null && item.isAlive()) {
                try {
                    return (T) item.getValue();
                } catch (ClassCastException e) {
                    return null;
                }
            } else {
                // 主动清理失效缓存
                cacheMap.remove(key);
            }
        }
        return null;
    }

    @Override
    public boolean clear() {
        cacheMap.clear();
        return cacheMap.isEmpty();
    }

    static class CacheItem {
        private long aliveUntil;
        private Object value;

        CacheItem(long aliveTime, Object value) {
            this.value = value;
            // 记录缓存存活到期时间，判断当前时间与此记录值，从而判定缓存是否过期
            this.aliveUntil = System.currentTimeMillis() + aliveTime;
        }

        public boolean isAlive() {
            return System.currentTimeMillis() < aliveUntil;
        }

        public Object getValue() {
            return value;
        }
    }
}
```

<br>

### 总结

<br>

仅少量数据需要缓存以提高系统性能时，可采用上述基于内存缓存实现方案。
也可使用此方案缓存热点数据

<br><br><br><br>