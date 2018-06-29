Cache模块使用了==装饰器模式==。

### 1.Cache接口

此接口定义了所有缓存的基本行为。

```java
public interface Cache {
  
  //取得ID
  String getId();

  //存入值
  void putObject(Object key, Object value);
  
  //获取值
  Object getObject(Object key);

  
  //删除值
  Object removeObject(Object key);
  
  //清空
  void clear();
 
  //取得大小
  int getSize();
  
  //取得读写锁, 从3.2.6开始没用了，要SPI自己实现锁
  ReadWriteLock getReadWriteLock();

}
```

### 2.实现类
#### 1.BlockingCache
这是一个阻塞缓存装饰器，在获取缓存时需要先获取对应key的锁。

如果获取不到对应的缓存会先阻塞等待一段时间。

#### 2.FifoCache
FIFO缓存器，固定了缓存的数量，并使用先进先出来淘汰多余缓存。

使用==linkedHashMap==来实现
#### 3.LoggingCache
日志缓存，取缓存时打印命中率信息

#### 4.LruCache
最近最少使用缓存，来淘汰多余缓存。

####5.ScheduledCache

定时调度缓存，目的是==每隔一段时间==(默认每一小时)清空一下缓存。

#### 6.SerializedCache

序列化缓存，用途是先将对象序列化成2进制，再缓存,好处是将对象压缩了，==省内存==；坏处是==速度慢==了

#### 7.SoftCache

软引用缓存,核心是SoftReference，目的是当value没用时使它失效。

#### 8.SynchronizedCache

同步缓存，防止多线程问题，核心: 加锁。

#### 9.TransactionalCache

事务缓存，一次性存入多个缓存，移除多个缓存。

#### 10.WeakCache

弱引用缓存，可以看到代码和SoftCache如出一辙，就是SoftReference变成了WeakReference。



### 3.基础知识

#### 1.linkedHashMap中的参数设置和删除功能的自定义

#### 2.java引用的使用，包括弱引用、软引用和其它引用

#### 3.装饰器模式

#### 4.锁的使用Lock

#### 5.HashMap中的putIfAbsent(key,value)函数

这个函数的功能就是：如果键值存在则返回这个，不存在则返回参数中的。

```java
default V putIfAbsent(K key, V value) {
        V v = get(key);
        if (v == null) {
            v = put(key, value);
        }
 
        return v;
    }
```



