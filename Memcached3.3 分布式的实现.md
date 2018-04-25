# Memcached分布式

memcached虽然称为“分布式”缓存服务器，但服务器端并没有“分布式”功能。至于memcached的分布式，则是完全由客户端程序库实现的。 这种分布式是memcached的最大特点。

下面假设memcached服务器有node1～node3三台， 应用程序要保存数据 {‘zone’:’nova’}。

![](https://raw.githubusercontent.com/wiki/liudongdongdong/temp/consistenthash1.png)

​									图1 分布式简介：准备

首先向memcached中添加“zone”。将“zone”传给客户端程序库后， 客户端实现的算法就会根据“键”来决定保存数据的memcached服务器。 服务器选定后，即命令它保存“zone”及其值。

![](https://raw.githubusercontent.com/wiki/liudongdongdong/temp/consistenthash2.png)

​									图2 分布式简介：添加时

接下来获取保存的数据。获取时也要将要获取的键“zone”传递给函数库。 函数库通过与数据保存时相同的算法，根据“键”选择服务器。 使用的算法相同，就能选中与保存时相同的服务器，然后发送get命令。 只要数据没有因为某些原因被删除，就能获得保存的值。

![图片3](https://raw.githubusercontent.com/wiki/liudongdongdong/temp/consistenthash3.png)

​									图3 分布式简介：获取时

这样，将不同的键保存到不同的服务器上，就实现了memcached的分布式。 memcached服务器增多后，键就会分散，即使一台memcached服务器发生故障 无法连接，也不会影响其他的缓存，系统依然能继续运行。

### 余数hash

在Memecache集群中，如何平均分配所有的key？一个自然的办法是用余数hash。

```
node_id = hash_key % len(nodes) 		#node_id 是机器的id。
```

随着负载的增加，现在线上要加一台机器。此时会出现什么情况呢？

假设原有机器3台，加入一台新机器后，会有4台机器。

原有3台机器的编号为0、1、2。当加入3号机后，将会有3/4的key不能命中原有的缓存机器。

分析如下：

对0号机而言，当有3台机器的时候，只接受被3整除的key。现在，0号机只接受被4整除的key。所以，只有被12整除的key被保留下来。这台机器原来接受1/3的key，其中1/4的key还在，所以只有 1/4 的key可以被保留。

对1号机也可以做类似的分析，留下来的必定是除12余1的key，因此也是1/4的key被保留。同理可以知道2号机的情况也是如此。

于是，只有1/4 的key被保留，也就是说3/4的key不能命中原有的机器。

事实上，我们可以写出key流失率的公式

```
miss_ratio = n/(n+1) 	# n 是机器的数目
```

如果原来有99台机器，新加入一台机器后，将有 1 - 1/100 = 99/100 的key不能保留。这些不命中缓存的压力将转嫁给数据库，使得数据库崩坏。

参考：[余数Hash和负载均衡](https://zhuanlan.zhihu.com/xiaochi/19786777)

### Consistent Hashing

Consistent Hashing如下所示：首先求出memcached服务器（节点）的哈希值， 并将其配置到0～232的圆（continuum）上。 然后用同样的方法求出存储数据的键的哈希值，并映射到圆上。 然后从数据映射到的位置开始顺时针查找，将数据保存到找到的第一个服务器上。 如果超过232仍然找不到服务器，就会保存到第一台memcached服务器上。

[![memcached-0004-04](https://xenojoshua.com/assets/memcached-0004-04.png)](https://xenojoshua.com/uploads/2011/04/memcached-0004-04.png)

图4 Consistent Hashing：基本原理

从上图的状态中添加一台memcached服务器。余数分布式算法由于保存键的服务器会发生巨大变化 而影响缓存的命中率，但Consistent Hashing中，只有在continuum上增加服务器的地点逆时针方向的 第一台服务器上的键会受到影响。

[![memcached-0004-05](https://xenojoshua.com/assets/memcached-0004-05.png)](https://xenojoshua.com/uploads/2011/04/memcached-0004-05.png)

​									图5 Consistent Hashing：添加服务器

因此，Consistent Hashing最大限度地抑制了键的重新分布。 而且，有的Consistent Hashing的实现方法还采用了虚拟节点的思想。 使用一般的hash函数的话，服务器的映射地点的分布非常不均匀。 因此，使用虚拟节点的思想，为每个物理节点（服务器） 在continuum上分配100～200个点。这样就能抑制分布不均匀， 最大限度地减小服务器增减时的缓存重新分布。

通过下文中介绍的使用Consistent Hashing算法的memcached客户端函数库进行测试的结果是， 由服务器台数（n）和增加的服务器台数（m）计算增加服务器后的命中率计算公式如下：
$$
(1 - n/(n+m)) * 100
$$

参考：http://tech.idv2.com/2008/07/24/memcached-004/



OpenStack使用python-memcached客户端，从server = self.buckets[serverhash % len(self.buckets)]可以看出，使用的是简单的余数hash算法

```python
class Client(threading.local):
    def _get_server(self, key):
        if isinstance(key, tuple):
            serverhash, key = key
        else:
            serverhash = serverHashFunction(key)

        if not self.buckets:
            return None, None

        for i in range(Client._SERVER_RETRIES):
            server = self.buckets[serverhash % len(self.buckets)]
            if server.connect():
                # print("(using server %s)" % server,)
                return server, key
            serverhash = str(serverhash) + str(i)
            if isinstance(serverhash, six.text_type):
                serverhash = serverhash.encode('ascii')
            serverhash = serverHashFunction(serverhash)
        return None, None
```
