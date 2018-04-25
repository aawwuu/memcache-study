

# Memcached数据回收机制

https://www.jianshu.com/p/049717570769


数据不会真正从Memcached消失
Memcached不会主动释放已分配的内存，记录超时后，客户端get该记录，Memcached不会返回该记录，并标志该记录过期失效，此时内存空间可重复使用。

### Lazy Expiration

Memcached内部不会监视记录是否过期，而是客户端get时查看记录的时间戳，检查记录是否过期，如果过期，标志为过期，下次存放新记录优先使用过期记录占用的内存，这种技术就是Lazy Expiration，Memcached不会在过期监控上耗费CPU资源。

### LRU: Least Recently Used

Memcached优先使用记录已超时和未使用的内存空间，但是在追加新记录时如果没有记录已超时和未使用的内存空间，此时使用Least Recently Used（LRU）机制分配空间。顾名思义，就是删除”最近最少使用“的记录的机制。当Memcached内存空间不足（无法从slab class获取Chunk）时，就删除”最近最少使用“的记录，将其内存空间分配给新记录存放。从缓存使用角度，该模型非常理想。

如果想关闭LRU机制，启动Memcached指定-M参数即可。