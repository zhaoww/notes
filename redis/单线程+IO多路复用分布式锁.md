#### 为什么Redis这么快？  
1.采用了多路复用io阻塞机制  
2.数据结构简单，操作节省时间  
3.运行在内存中，自然速度快  
4.单线程避免了不必要的上下文切换和竞争条件

因为Redis的瓶颈不是cpu的运行速度，而往往是网络带宽和机器的内存大小。再说了，单线程切换开销小，容易实现既然单线程容易实现，而且CPU不会成为瓶颈，那就顺理成章地采用单线程的方案了


#### 多路 I/O 复用模型
多路 I/O 复用模型是利用select、poll、epoll可以同时监察多个流的 I/O 事件的能力，在空闲的时候，会把当前线程阻塞掉，当有一个或多个流有I/O事件时，就从阻塞态中唤醒，于是程序就会轮询一遍所有的流（epoll是只轮询那些真正发出了事件的流），并且只依次顺序的处理就绪的流，这种做法就避免了大量的无用操作。这里“多路”指的是多个网络连接，“复用”指的是复用同一个线程。采用多路 I/O 复用技术可以让单个线程高效的处理多个连接请求（尽量减少网络IO的时间消耗），且Redis在内存中操作数据的速度非常快（内存内的操作不会成为这里的性能瓶颈），主要以上两点造就了Redis具有很高的吞吐量。

#### 指令的区别  
select  
1.select 如果任何一个sock(I/O stream)出现了数据，select 仅仅会返回，但是并不会告诉你是那个sock上有数据，于是你只能自己一个一个的找，10几个sock可能还好，要是几万的sock每次都找一遍  
2.select 只能监视1024个链接,linux 定义在头文件中的，参见FD_SETSIZE。  
3.select 不是线程安全的  

poll
去除了1024的限制

epoll
epoll 现在是线程安全的。
epoll 现在不仅告诉你sock组里面数据，还会告诉你具体哪个sock有数据，你不用自己去找了。 


#### keys代替方案
找到前缀是ABC的所有KEYS,时间复杂度O(N)。可以使用，但是在生产环境中，这么使用肯定是不行的，因为生产环境的key的数量比较多，一次查询会block其他操作。而更重要的是一次性返回这么多的key，数据量比较大，网络传输成本高。所以一般生产环境中去找符合某些条件的KEYS一般使用SCAN 或 Sets。
集合来操作比较好理解，一个个的pop出来，但是相当于在原有的数据结构上多了一个keys的set集合
```java
/**
 * 获取符合条件的key（scan替代keys）
 *
 * @param pattern 表达式
 * @return list
 */
public Set<String> keys(String pattern) {
    Set<String> keys = Sets.newHashSet();
    this.scan(pattern, item -> keys.add(new String(item, StandardCharsets.UTF_8)));
    return keys;
}

/**
 * 扫描key
 *
 * @param pattern  表达式
 * @param consumer consumer
 */
private void scan(String pattern, Consumer<byte[]> consumer) {
    lettuceRedisTemplate.execute((RedisConnection connection) -> {
        try (Cursor<byte[]> cursor = connection.scan(ScanOptions.scanOptions().count(CURSOR_LIMIT)
                .match(pattern).build())) {
            cursor.forEachRemaining(consumer);
            return null;
        } catch (IOException e) {
            log.warn("扫描keys异常, pattern:{}", pattern, e);
            throw new RuntimeException(e);
        }
    });
}
```