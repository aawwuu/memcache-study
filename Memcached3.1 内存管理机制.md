# Memcached内存管理机制

Memcached使用Slab Allocation机制管理内存，基本原理是按照预先规定的大小，将分配的内存分割成特定长度的块， 以完全解决内存碎片问题。并且Slab Allocator还有重复使用已分配的内存的目的，也就是说，分配到的内存不会释放，而是重复利用。

![img](https://xenojoshua.com/assets/memcached-0002-01.png)

​											*Slab Allocation*

### Slab Allocation的主要术语

**Page为内存分配的单位**

分配给Slab的内存空间，默认是1MB，可以通过-I参数修改。分配给Slab之后根据slab的大小切分成chunk。

如果需要申请内存时，memcached会划分出一个新的page并分配给需要的slab区域。page一旦被分配在memcached重启前不会被回收或者重新分配。

**Chunk为存放缓存数据的单位**

Chunk是一系列固定的内存空间，它的大小和包含它的slab的大小是一致的。如果实际的数据大小小于chunk的大小，空余的空间将会被闲置，这个是为了防止内存碎片而设计的。

**Slabs划分数据空间**

Slab是memcached用来划定存储空间的大小概念，每当memcached启动的时候，它会按照-n参数配置的值来决定第一个slab的大小，然后根据-f参数的值来决定后续slab大小的增长速率，一个一个地决定后续的slab的大小，直到slab的大小达到设定的page大小（一般是1M）。有相同大小chunk的slab被组织在一起，称为slab_class。

![img](https://upload-images.jianshu.io/upload_images/6060459-dfabac0d72180b4d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/700)



## 在Slab中缓存记录的原理

下面说明memcached如何针对客户端发送的数据选择slab并缓存到chunk中。

memcached根据收到的数据的大小，选择最适合数据大小的slab。 memcached中保存着slab内空闲chunk的列表，根据该列表选择chunk， 然后将数据缓存于其中。

[![memcached-0002-02](https://xenojoshua.com/assets/memcached-0002-02.png)](https://xenojoshua.com/uploads/2011/04/memcached-0002-02.png)

## Slab Allocator的缺点

Slab Allocator解决了当初的内存碎片问题，但新的机制也给memcached带来了新的问题。

这个问题就是，由于分配的是特定长度的内存，因此无法有效利用分配的内存。 例如，将100字节的数据缓存到128字节的chunk中，剩余的28字节就浪费了。

[![memcached-0002-03](https://xenojoshua.com/assets/memcached-0002-03.png)](https://xenojoshua.com/uploads/2011/04/memcached-0002-03.png)

对于内存浪费问题目前还没有完美的解决方案。

比较有效的方式是，如果预先知道客户端发送的数据的公用大小，或者仅缓存大小相同的数据的情况下， 只要使用适合数据大小的组的列表，就可以减少浪费。

## 使用Growth Factor进行调优

Memcached启动时会指定增长因子，默认是1.25

初始化slabclass，用于维护所有slab class信息，默认情况下一次性初始化为200个，并且class的最大数量也是200个。  计算每个class中的chunk尺寸，计算每个page可以容纳的chunk数量，也就是perslab（每页可容纳的chunk数量），如果-I指定的大小为1M，chunk的尺寸为500K，那么perslab的值是2，如果chunk的尺寸是600K或700K，那么perslab的值只能是1。  在初始化的过程当中，并没有为每个class预分配任何内容，只是预算了不同class的结构信息，除非你去掉了源代码中的DONT_PREALLOC_SLABS宏定义，这时候在初始化的时候就会为每个class先分配16页的空间

```
[root@172-28-14-43 ~]# memcached -u memcached -p 18888 -m 64 -c 1024 -l 10.10.11.43 -vv
slab class   1: chunk size        96 perslab   10922
slab class   2: chunk size       120 perslab    8738
slab class   3: chunk size       152 perslab    6898
slab class   4: chunk size       192 perslab    5461
slab class   5: chunk size       240 perslab    4369
slab class   6: chunk size       304 perslab    3449
slab class   7: chunk size       384 perslab    2730
slab class   8: chunk size       480 perslab    2184
slab class   9: chunk size       600 perslab    1747
slab class  10: chunk size       752 perslab    1394
slab class  11: chunk size       944 perslab    1110
slab class  12: chunk size      1184 perslab     885
slab class  13: chunk size      1480 perslab     708
slab class  14: chunk size      1856 perslab     564
```

指定增长因子为2：

```
[root@172-28-14-43 ~]# memcached -u memcached -p 18888 -m 64 -c 1024 -l 10.10.11.43 -f 2 -vv
slab class   1: chunk size        96 perslab   10922
slab class   2: chunk size       192 perslab    5461
slab class   3: chunk size       384 perslab    2730
slab class   4: chunk size       768 perslab    1365
slab class   5: chunk size      1536 perslab     682
slab class   6: chunk size      3072 perslab     341
slab class   7: chunk size      6144 perslab     170
slab class   8: chunk size     12288 perslab      85
slab class   9: chunk size     24576 perslab      42
slab class  10: chunk size     49152 perslab      21
slab class  11: chunk size     98304 perslab      10
slab class  12: chunk size    196608 perslab       5
slab class  13: chunk size    524288 perslab       2
```

将memcached引入产品，或是直接使用默认值进行部署时， 最好是重新计算一下数据的预期平均长度，调整growth factor， 以获得最恰当的设置。内存是珍贵的资源，浪费就太可惜了。

**Memcached内存管理需要注意的几个方面：**

- chunk是在page里面划分的，而page固定为1m，所以chunk最大不能超过1m。
- chunk实际占用内存要加48B，因为chunk数据结构本身需要占用48B。
- 如果用户数据大于1m，则memcached会将其切割，放到多个chunk内。
- 已分配出去的page不能回收。

> **对于key-value信息，最好不要超过1m的大小；同时信息长度最好相对是比较均衡稳定的，这样能够保障最大限度的使用内存；同时，memcached采用的LRU清理策略，合理甚至过期时间，提高命中率。**

无特殊场景下，key-value能满足需求的前提下，使用memcached分布式集群是较好的选择，搭建与操作使用都比较简单；分布式集群在单点故障时，只影响小部分数据异常，目前还可以通过Magent缓存代理模式，做单点备份，提升高可用；整个缓存都是基于内存的，因此响应时间是很快，不需要额外的序列化、反序列化的程序，但同时由于基于内存，数据没有持久化，集群故障重启数据无法恢复。高版本的memcached已经支持CAS模式的原子操作，可以低成本的解决并发控制问题。

参考：[缓存那些事](https://tech.meituan.com/cache_about.html)
