

# Memcached工作过程

https://www.cnblogs.com/kuzhon/articles/5370237.html

https://xenojoshua.com/2011/04/memcached-anatomy-index/

https://www.jianshu.com/p/049717570769

http://blog.csdn.net/wwd0501/article/details/46851119



![img](https://upload-images.jianshu.io/upload_images/6060459-cb410114782e3e39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/656)

同时基于这张图，理一下MemCache一次写缓存的流程：

1、应用程序输入需要写缓存的数据

2、API将Key输入路由算法模块，路由算法根据Key和MemCache集群服务器列表得到一台服务器编号

3、由服务器编号得到MemCache及其的ip地址和端口号

4、API调用通信模块和指定编号的服务器通信，将数据写入该服务器，完成一次分布式缓存的写操作

读缓存和写缓存一样，只要使用相同的路由算法和服务器列表，只要应用程序查询的是相同的Key，MemCache客户端总是访问相同的客户端去读取数据，只要服务器中还缓存着该数据，就能保证缓存命中。



### Memcached保存记录过程

图8为Memcached保存item流程图，具体步骤为：

第一步，从LRU队列寻找过期item，这里的LRU队列是相同Slab一个队列，而不是全局统一，过期item标记方法通过Lazy Expiration实现，如果有过期的item，使用新item替换过期item，结束；
第二步，如果没有过期item，查看是否有合适空闲的Chunk，如果有，保存新item到空闲Chunk，结束；
第三步，如果没有合适空闲的Chunk，尝试初始化一个新同等Chunk size的Slab，检查内存是否足够，如果够，分配内存创建Slab和Chunk，并使用Chunk存放新item，结束；
第四步，如果内存不够，从LRU队列淘汰最近最少使用的item，然后用这个Chunk存放新item，结束。注意这一步将导致非过期LRU数据丢失。

![img](https://upload-images.jianshu.io/upload_images/3190591-176bd63860e4e671.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)

​							图8 Memcached保存记录过程








