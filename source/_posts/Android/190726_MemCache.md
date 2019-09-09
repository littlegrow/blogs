---
title: 客户端支持带过期时间的缓存
date: 2019-7-26 9:57:00
description: "复杂列表下发时，考虑到接口性能，服务端并不会将全部数据封装返回，客户端拿到列表后按需请求加载，当用户快速滑入滑出列表时，将伴有大量重复请求发出，图片闪烁显示，体验很不友好。"
categories:
- 缓存
- Kotlin
---

### 背景

<br>

复杂列表下发时，考虑到接口性能，服务端并不会将全部数据封装返回，客户端拿到列表后按需请求加载，当用户快速滑入滑出列表时，伴有大量重复请求发出，图片频繁闪烁，体验很不友好。

另外这些数据可能具有时效性，比如商品价格可能随着活动开启或结束动态调整。

<br><br>

### 灵感来源

<br>

后端开发工程师常会用到[Memcached](https://memcached.org/)缓存，这是一种基于内存的key-value存储，用来存储小块的任意数据，支持设置缓存有效时间。

基于以上业务场景和对Memcached的理解，在客户端实现类似缓存会很有价值。

<br><br>

### 实现方式

<br>

提及客户端缓存用的最多的是LruCache，先来了解一下，下面注释片段摘自 Android sdk LruCache 源码:

```
/**
 - A cache that holds strong references to a limited number of values. Each time
 - a value is accessed, it is moved to the head of a queue. When a value is
 - added to a full cache, the value at the end of that queue is evicted and may
 - become eligible for garbage collection.
 
 ......

 - This class appeared in Android 3.1 (Honeycomb MR1)
 */
```

LruCache是从 Android 3.1 开始提供的缓存类，用来实现内存缓存很合适；
LRU采用近期最少使用算法，当缓存满时， 优先淘汰最近最少使用的缓存对象。 具体实现可查看[开发者官网-LRU](https://developer.android.com/reference/kotlin/android/util/LruCache?hl=en)

<br><br>

#### 缓存数据结构

<br>

```
import java.lang.ref.SoftReference

/**
 * 缓存 item
 *
 * 使用软引用持有缓存，防止内存紧张时无法回收
 * 添加 aliveTime 标记缓存到期时间
 *
 * @author littlegrow
 */
class CacheItem<K, V>(val key: K, value: V, val aliveTime: Long) {
    val valueRef: SoftReference<V> = SoftReference(value)
}
```

<br><br>

#### 缓存实现

<br>

```
import android.util.LruCache

/**
 * 使用LruCache实现支持过期时间的缓存
 *
 * @author littlegrow
 */

class MemCache<K, V>(maxSize: Int) {
    companion object {
        // 默认有效时间5分钟
        private const val DEFAULT_DURING = (5 * 60 * 1000).toLong()
    }

    private val lruCache: LruCache<K, CacheItem<K, V>> = LruCache(maxSize)

    /**
     * 根据key获取缓存
     *
     * @param key key
     * @return 缓存不存在或者失效返回 null
     */
    fun get(key: K): V? {
        val cacheItem = lruCache.get(key) ?: return null
        return if (isCacheItemAlive(cacheItem)) {
            // 缓存内容被回收则移除缓存
            if (cacheItem.valueRef.get() == null) {
                remove(key)
            }
            cacheItem.valueRef.get()
        } else {
            lruCache.remove(key)
            null
        }
    }

    /**
     * 设置缓存，支持自定义缓存存活时间
     *
     * @param key key
     * @param value value
     * @param aliveTime 存活时间
     * @return 如果缓存存在更新缓存时间，并返回缓存值，否则返回 null
     */
    @JvmOverloads
    fun put(key: K, value: V, aliveTime: Long = DEFAULT_DURING): V? {
        if (aliveTime < 0) {
            throw IllegalArgumentException("aliveTime should >= 0")
        }
        val time = System.currentTimeMillis()
        val cacheItem = CacheItem(key, value, time + aliveTime)
        val previous = lruCache.put(key, cacheItem)
        return previous?.valueRef?.get()
    }

    /**
     * 移除缓存
     *
     * @param key key
     * @return 如果有缓存则返回缓存内容，否则返回 null
     */
    fun remove(key: K): V? {
        return lruCache.remove(key)?.valueRef?.get()
    }

    /**
     * 判断缓存是否存活
     */
    private fun isCacheItemAlive(cacheItem: CacheItem<K, V>): Boolean {
        return System.currentTimeMillis() <= cacheItem.aliveTime
    }
}
```

<br><br>

### 效果

<br>

应用此缓存后，列表滑动过程中减少了大量的无效请求，杜绝了图片加载闪烁的问题。

如果您有类似业务场景，可考虑采用。

<br><br><br><br><br>