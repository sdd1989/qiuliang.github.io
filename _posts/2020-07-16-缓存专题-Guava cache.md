---
layout: post
title:  "缓存专题-Guava cache"
tags:   Java
date:   2020-07-06 16:30:00 +0800-- 
categories: [工具] 
---
# **缓存专题-Guava cache**

[TOC]

## 1、缓存策略（广义）

### 1、Cache Aside（旁路策略）

![image-20200715210240566](/assets/imgs/image-20200715210240566.png)

正确姿势如下步骤

**读取流程：**

从缓存中读取数据；如果缓存命中，则直接返回数据；如果缓存不命中，则从数据库中查询数据；查询到数据后，将数据写入到缓存中，并且返回给用户。

**更新流程：**

更新数据库中的记录；删除缓存记录。



下面是一些可能出现的一些一致性问题：

![image-20200715210839587](/assets/imgs/image-20200715210839587.png)

​																		先更新数据库，再更新缓存

![image-20200715210712994](/assets/imgs/image-20200715210712994.png)

​																	先删除缓存，再更新数据库

​																																															

![image-20200715210555077](/assets/imgs/image-20200715210555077.png)

​									           先更新数据库，删除缓存（此问题出现概率很低，因为缓存速度远远大于数据库）



Cache Aside 是我们在使用分布式缓存时最常用的策略，你可以在实际工作中直接拿来使用。

### 2、Read/Write Through（读穿/写穿）策略

![image-20200715210959600](/assets/imgs/image-20200715210959600.png)

​																				guava cache是此策略

Read/Write Through 和 Write Back 策略需要缓存组件的支持，所以比较适合你在实现本地缓存组件的时候使用

### 3、Write Back/Cache behind（写回）策略

![image-20200715211030406](/assets/imgs/image-20200715211030406.png)

![image-20200715211040016](/assets/imgs/image-20200715211040016.png)

Write Back 策略是计算机体系结构中的策略，不过写入策略中的只写缓存，异步写入后端存储的策略倒是有很多的应用场景。

比如，MySQL写数据，先写入buffer，和undo log，redo log，异步刷磁盘（sync）

## 2、数据结构

![image-20200715201306589](/assets/imgs/image-20200715201306589.png)

![image-20200715201436282](/assets/imgs/image-20200715201436282.png)

![image-20200716112250095](/assets/imgs/image-20200716112250095.png)

![image-20200716112502011](/assets/imgs/image-20200716112502011.png)

```java
final Queue<ReferenceEntry<K, V>> recencyQueue //命中时写入，写操作或者淘汰的时候清空
final Queue<ReferenceEntry<K, V>> writeQueue //按写入时间排序，元素在写入时加入队列尾部，元素过期或淘汰的时候移除
final Queue<ReferenceEntry<K, V>> accessQueue //按访问时间排序，元素在访问时加入队列尾部，元素过期或淘汰的时候移除

```

![image-20200716115242205](/assets/imgs/image-20200716115242205.png)writeQueue，accessQueue是双向循环队列（链表）

****

##  3、加载

### 1、From a CacheLoader

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
                .maximumSize(2)
                .build(new CacheLoader<String, String>() {
                    @Override
                    public String load(String key) { // no checked exception
                        return createExpensiveGraph(key);
                    }
                });
```

### 2、From a Callable

```java
Cache<Key, Value> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .build(); // look Ma, no CacheLoader
...
try {
  // If the key wasn't in the "easy to compute" group, we need to
  // do things the hard way.
  cache.get(key, new Callable<Value>() {
    @Override
    public Value call() throws AnyException {
      return doThingsTheHardWay(key);
    }
  });
} catch (ExecutionException e) {
  throw new OtherException(e.getCause());
}
```

### 3、显式put

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
                .maximumSize(2)
                .build();
cache.put("key1", "value1");
```



## 4、缓存回收

### 1、基于时间

- **refreshAfterWrite**：当缓存项上一次更新操作之后的多久会被刷新。

- **expireAfterWrite**：当缓存项在指定的时间段内没有更新就会被回收。

- **expireAfterAccess**：当缓存项在指定的时间段内没有被读或写就会被回收

- **refreshAfterWrite vs expireAfterWrite**：

  expireAfterWrite：

  到期之后只有一个线程去load，其他阻塞

  load完成后，阻塞线程轮流获取锁，获取值，释放锁，性能差

  refreshAfterWrite：

  到期之后只有一个线程load，其他线程不会阻塞，返回旧值，对数据一致性要求高的程序可能会有问题，性能好

- **源码分析**

  ```java
  
  //refreshAfterWrite过期加载
  V scheduleRefresh(
          ReferenceEntry<K, V> entry,
          K key,
          int hash,
          V oldValue,
          long now,
          CacheLoader<? super K, V> loader) {
        if (map.refreshes()
            && (now - entry.getWriteTime() > map.refreshNanos)//判断过期
            && !entry.getValueReference().isLoading()) {
          V newValue = refresh(key, hash, loader, true);
          if (newValue != null) {
            return newValue;
          }
        }
        return oldValue;
  }
  
  
  //expireAfterWrite过期加载
  V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
        checkNotNull(key);
        checkNotNull(loader);
        try {
          if (count != 0) { // read-volatile
            // don't call getLiveEntry, which would ignore loading values
            ReferenceEntry<K, V> e = getEntry(key, hash);
            if (e != null) {
              long now = map.ticker.read();
              V value = getLiveValue(e, now);
              if (value != null) {
                recordRead(e, now);//如果是expiresAfterAccess模式，记录读取时间，加入recencyQueue
                statsCounter.recordHits(1);//记录命中数
                return scheduleRefresh(e, key, hash, value, now, loader);
              }
              ValueReference<K, V> valueReference = e.getValueReference();
              if (valueReference.isLoading()) {
                return waitForLoadingValue(e, key, valueReference);
              }
            }
          }
  
          // at this point e is either null or expired;
          return lockedGetOrLoad(key, hash, loader);
        } catch (ExecutionException ee) {
          Throwable cause = ee.getCause();
          if (cause instanceof Error) {
            throw new ExecutionError((Error) cause);
          } else if (cause instanceof RuntimeException) {
            throw new UncheckedExecutionException(cause);
          }
          throw ee;
        } finally {
          postReadCleanup();
        }
      }
  
  		/**
       * Gets the value from an entry. Returns null if the entry is invalid, partially-collected,
       * loading, or expired.
       */
      V getLiveValue(ReferenceEntry<K, V> entry, long now) {
        if (entry.getKey() == null) {
          tryDrainReferenceQueues();
          return null;
        }
        V value = entry.getValueReference().get();
        if (value == null) {
          tryDrainReferenceQueues();
          return null;
        }
  
        if (map.isExpired(entry, now)) {
          tryExpireEntries(now);//过期
          return null;
        }
        return value;
      }
  
  /** Returns true if the entry has expired. */
    boolean isExpired(ReferenceEntry<K, V> entry, long now) {
      checkNotNull(entry);
      if (expiresAfterAccess() && (now - entry.getAccessTime() >= expireAfterAccessNanos)) {
        return true;
      }
      if (expiresAfterWrite() && (now - entry.getWriteTime() >= expireAfterWriteNanos)) {
        return true;
      }
      return false;
    }
  
  // expiration
  
      /** Cleanup expired entries when the lock is available. */
      void tryExpireEntries(long now) {
        if (tryLock()) {
          try {
            expireEntries(now);
          } finally {
            unlock();
            // don't call postWriteCleanup as we're in a read
          }
        }
      }
  
      @GuardedBy("this")
      void expireEntries(long now) {
        drainRecencyQueue();//清空recencyQueue，当accessQueue队列有此元素的话，移到队尾accessQueue
  
        ReferenceEntry<K, V> e;
        while ((e = writeQueue.peek()) != null && map.isExpired(e, now)) {
          if (!removeEntry(e, e.getHash(), RemovalCause.EXPIRED)) {
            throw new AssertionError();
          }
        }
        while ((e = accessQueue.peek()) != null && map.isExpired(e, now)) {
          if (!removeEntry(e, e.getHash(), RemovalCause.EXPIRED)) {
            throw new AssertionError();
          }
        }
      }
  ```

  

### 2、基于容量

LRU淘汰策略

```java
//LRU
void evictEntries(ReferenceEntry<K, V> newest) {
      if (!map.evictsBySize()) {
        return;
      }

      drainRecencyQueue();//清空recencyQueue，当accessQueue队列有此元素的话，移到队尾accessQueue

      // If the newest entry by itself is too heavy for the segment, don't bother evicting
      // anything else, just that
      if (newest.getValueReference().getWeight() > maxSegmentWeight) {
        if (!removeEntry(newest, newest.getHash(), RemovalCause.SIZE)) {
          throw new AssertionError();
        }
      }

      while (totalWeight > maxSegmentWeight) {
        ReferenceEntry<K, V> e = getNextEvictable();
        if (!removeEntry(e, e.getHash(), RemovalCause.SIZE)) {
          throw new AssertionError();
        }
      }
    }


ReferenceEntry<K, V> getNextEvictable() {
  for (ReferenceEntry<K, V> e : accessQueue) {
    int weight = e.getValueReference().getWeight();
    if (weight > 0) {
      return e;
    }
  }
  throw new AssertionError();
}
```



### 3、基于引用

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
                .weakKeys()
                .weakValues()
                .softValues()
                .build(new CacheLoader<String, String>() {
                    @Override
                    public String load(String key) { // no checked exception
                        return loadBy(key);
                    }
                });
```



### 4、显式清除

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
                .maximumSize(2)
                .build(new CacheLoader<String, String>() {
                    @Override
                    public String load(String key) { // no checked exception
                        return createExpensiveGraph(key);
                    }
                });
        cache.getUnchecked("key1");
        cache.getUnchecked("key2");
        System.out.println(cache.asMap());
        //cache.invalidate("key1");
        //cache.invalidateAll(Lists.newArrayList("key1","key2"));
        cache.invalidateAll();
        System.out.println(cache.asMap());
```



### 5、移除监听器

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
                .maximumSize(2)
                .refreshAfterWrite(1,TimeUnit.SECONDS)
                .removalListener(new RemovalListener<String, String>() {

                    @Override
                    public void onRemoval(RemovalNotification<String, String> notification) {
                        System.out.println(notification + ":" + notification.getCause());
                    }
                })
                .build(new CacheLoader<String, String>() {
                    @Override
                    public String load(String key) { // no checked exception
                        return createExpensiveGraph(key);
                    }
                });

        cache.getUnchecked("key1");
        cache.getUnchecked("key2");
        cache.getUnchecked("key3");
        TimeUnit.SECONDS.sleep(2);
        cache.getUnchecked("key2");
```

```
current thread:main createExpensiveGraph,key=key1
current thread:main createExpensiveGraph,key=key2
current thread:main createExpensiveGraph,key=key3
key1=value1:SIZE
current thread:main createExpensiveGraph,key=key2
key2=value2:REPLACED
```



## 5、其他特性

### 1、统计

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
                .maximumSize(2)
                .refreshAfterWrite(1,TimeUnit.SECONDS)
                .recordStats()
                .build(new CacheLoader<String, String>() {
                    @Override
                    public String load(String key) { // no checked exception
                        return createExpensiveGraph(key);
                    }
                });
        cache.getUnchecked("key1");
        cache.getUnchecked("key2");
        cache.getUnchecked("key3");
        TimeUnit.SECONDS.sleep(2);
        cache.getUnchecked("key2");
        System.out.println("Stats:" + cache.stats());
```

```
current thread:main createExpensiveGraph,key=key1
current thread:main createExpensiveGraph,key=key2
current thread:main createExpensiveGraph,key=key3
current thread:main createExpensiveGraph,key=key2
Stats:CacheStats{hitCount=1, missCount=3, loadSuccessCount=4, loadExceptionCount=0, totalLoadTime=15623047, evictionCount=1}
```



### 2、asMap视图

```java
LoadingCache<String, String> cache = CacheBuilder.newBuilder()
                .maximumSize(2)
                .build(new CacheLoader<String, String>() {
                    @Override
                    public String load(String key) { // no checked exception
                        return createExpensiveGraph(key);
                    }
                });
        System.out.println(cache.asMap());
```



## 6、替代品Caffeine cache

![preview](https://pic4.zhimg.com/v2-78569d5b83551cd189512e9344e3c8f5_r.jpg)

- 性能更高

- Disruptor（Ringbuffer）

- 异步加载

- W-TinyLFU（分段LRU算法）和MySQL Buffer Pool的newList，oldList有点像

  ![preview](https://pic3.zhimg.com/v2-10435e4f87d9a698daa215d0bc163da0_r.jpg)

  对于长期保留的数据，W-TinyLFU使用了分段LRU（Segmented LRU，缩写SLRU）策略。起初，一个数据项存储被存储在试用段（probationary segment）中，在后续被访问到时，它会被提升到保护段（protected segment）中（保护段占总容量的80%）。保护段满后，有的数据会被淘汰回试用段，这也可能级联的触发试用段的淘汰。这套机制确保了访问间隔小的热数据被保存下来，而被重复访问少的冷数据则被回收

  ![preview](https://pic2.zhimg.com/v2-596898f382810fff75ed560bcce9121d_r.jpg)

- 使用，接口适配Guava cache，可以用guava接口操作

  ```XML
  				<dependency>
              <groupId>com.github.ben-manes.caffeine</groupId>
              <artifactId>caffeine</artifactId>
              <version>2.8.5</version>
          </dependency>
          <dependency>
              <groupId>com.github.ben-manes.caffeine</groupId>
              <artifactId>guava</artifactId>
              <version>2.8.5</version>
          </dependency>
  ```

  ```java
  LoadingCache<String, String> graphs = Caffeine.newBuilder()
                  .maximumSize(10000)
                  .expireAfterWrite(5, TimeUnit.SECONDS)
                  .refreshAfterWrite(1, TimeUnit.SECONDS)
                  .build(key -> createExpensiveGraph(key));
          graphs.get("key1");
          TimeUnit.SECONDS.sleep(1);
          graphs.get("key1");
          TimeUnit.SECONDS.sleep(4);
          graphs.get("key1");
  ```

  ```
  current thread:main createExpensiveGraph,key=key1
  current thread:ForkJoinPool.commonPool-worker-1 createExpensiveGraph,key=key1
  current thread:ForkJoinPool.commonPool-worker-1 createExpensiveGraph,key=key1
  ```

  ```java
  // Guava's LoadingCache interface
          com.google.common.cache.LoadingCache <String, String> graphs = CaffeinatedGuava.build(
                  Caffeine.newBuilder().maximumSize(10000)
                          .expireAfterWrite(5, TimeUnit.SECONDS)
                          .refreshAfterWrite(1, TimeUnit.SECONDS),
                  new CacheLoader<String, String>() { // Guava's CacheLoader
                      @Override public String load(String key) throws Exception {
                          return createExpensiveGraph(key);
                      }
                  });
          graphs.getUnchecked("key1");
          TimeUnit.SECONDS.sleep(1);
          graphs.getUnchecked("key1");
          TimeUnit.SECONDS.sleep(4);
          graphs.getUnchecked("key1");
  ```

  ```
  current thread:main createExpensiveGraph,key=key1
  current thread:ForkJoinPool.commonPool-worker-1 createExpensiveGraph,key=key1
  current thread:ForkJoinPool.commonPool-worker-1 createExpensiveGraph,key=key1
  ```

  **guava cache迁移到caffeine cache**

