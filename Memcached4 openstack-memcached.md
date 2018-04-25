参考：http://www.99cloud.net/html/2016/jiuzhouyuanchuang_0617/178.html

### 存储函数执行结果

其优势在于，很多查询函数的结果是固定的，但是又比较常用。查询一次之后，按照key-value存到cache中，再次查询时不需要访问数据库，直接从内存缓存中根据key将结果取出，可以提高很多速度。或者还有一些查询请求，查询的参数千变万化，但是结果只有固定的几类，这种更适合使用cache加速。

nova.availability_zones中查询虚拟机所属的availability zone

```python
AZ_CACHE_SECONDS = 60 * 60
# zone 更新很少, 所以缓存时间设为1小时，减少数据库的访问
def get_instance_availability_zone(context, instance):
    """Return availability zone of specified instance."""
    host = instance.host if 'host' in instance else None
    # 获取虚拟机所在的host
    if not host:
      	az = instance.get('availability_zone')
        #如果虚拟机还没有被分配到主机，就获取创建虚拟机时指定的zone
        return az

    cache_key = _make_cache_key(host)
    # 虚拟机的zone就是所在主机的zone，所以用host信息生成key
    cache = _get_cache()
    # 建立一个连接cache的client
    az = cache.get(cache_key)
    # 获取cache_key的value
    az_inst = instance.get('availability_zone')
    # 获取虚拟机的availability_zone属性
    if az_inst is not None and az != az_inst:
    # 两者对比，如果不一致,说明缓存中的信息是错误的，需要重新缓存。这种情况发生在虚拟机还没分配到有效主机之前，已经填充了缓存。
        az = None
    if not az:
        elevated = context.elevated()
        az = get_host_availability_zone(elevated, host)
        cache.set(cache_key, az)
        # 添加正确的缓存
    return az
```

第一次查询到这台主机上的虚拟机后，就会将zone信息缓存，以后再查询这台主机上的所有虚拟机的zone，都不会访问数据库。 这里支持的cache后端包括memcached,redis,mongondb或者是python的dict.目前主流openstack发行版推荐的选项是memcached。

### 存储token

OpenStack使用python-memcached模块作为memcached的客户端，python-memcached模块原生支持集群操作，其原理是在内存中维护一个主机列表，且集群中主机的权重值和主机在列表中重复出现的次数成正比。

token本身创建之后不会被修改，只会被读，所以很适合放到cache中，加速访问，也不需要写脚本定期清理数据库中的过期token。如果将token存放在数据库中，需要定期清理过期token，防止token表太大影响性能。

memcache_pool 在memcache的基础上，实现了memcache的连接池，可以在线程之间复用连接。

controller  /etc/nova/nova.conf

```shell
[keystone_authtoken]
memcached_servers = 10.131.73.127:18888,10.131.73.128:18888,10.131.73.129:18888
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = NOVA_PASS

[cache]
enabled = true
backend = oslo_cache.memcache_pool
memcache_servers = 10.131.73.127:18888,10.131.73.128:18888,10.131.73.129:18888
```

memcache driver的缺点在于，memcache本身是分布式的设计，但是并不是高可用的，如果一个memcached 服务发生重启，这个节点上的所有token都会丢失掉，会带来一些错误。
比这更困难的是，如果controller1网络不可达或者宕机，那么我们会观察到几乎每个openstack api请求都会有3s以上的卡顿。这是因为openstack默认使用python-memcached访问memcache，它提供的操作keystone的client继承自Thread.local类，和构建它的线程绑定。openstack服务启动后，会启动一定数量的子进程，每个http request到达，会有一个子进程接收，孵化一个线程去处理这个请求。如果用到memcache，线程会调用python-memcached 构建一个client类，通过这个client的实例对memcache进行操作。如果访问到网络不可达的memcache节点，卡住，操作超时，将这个节点标识为30秒内不可用，在这个线程内，不会再因此而卡住，但是这个http请求结束之后，下一个http请求过来，重新孵化的线程会reinit这个client，新的client丢失了旧的client的状态，还是可能会访问到卡住的memcache节点上。
社区之所以要做memcache_pool，就是为了解决这个问题，将client统一由pool管理起来，memcache节点的状态，也由pool管理起来，这样每个子进程里只会卡第一次，所以强烈推荐使用memcache_pool驱动而不是memcache。社区将memcache_pool的代码从keystone复制到oslo_cache项目中，希望所有使用memcache的项目都通过它的memcachepool去访问，避免这个问题。其中，nova在M版本支持，heat在L版本支持。
具体的各个服务如何配置使用memcache_pool driver这里不再赘述。



### 存储keystonemiddleware token

我们请求任何openstack服务时，该服务都要校验请求中提供的token是否合理，这部分代码显然不是每个项目都自己实现一遍，它在keystonemiddleware项目实现，并作为filter配置在各个项目的api-paste.ini文件中,如下所示:
[filter:authtoken]
paste.filter_factory = keystonemiddleware.auth_token:filter_factory
当请求到达服务时，由keystonemiddleware访问keystone来查询请求携带的token是否合法，通常我们一个token会使用很多次，所以keystonemiddleware建议使用memcache缓存，把从keystone取到的token缓存一段时间，默认是300秒，以减少对keystone的压力，提高性能。kolla项目中nova keystonemiddleware配置示例如下:
[keystone_authtoken]
[keystone_authtoken]
auth_uri = {{ internal_protocol }}://{{ kolla_internal_fqdn }}:{{ keystone_public_port }}
auth_url = {{ admin_protocol }}://{{ kolla_internal_fqdn }}:{{ keystone_admin_port }}
auth_plugin = password
project_domain_id = default
user_domain_id = default
project_name = service
username = {{ nova_keystone_user }}
password = {{ nova_keystone_password }}

memcache_security_strategy = ENCRYPT
memcache_secret_key = {{ memcache_secret_key }}
memcached_servers = {% for host in groups['memcached'] %}{{ hostvars[host]['ansible_' + hostvars[host]['api_interface']]['ipv4']['address'] }}:{{ memcached_port }}{% if not loop.last %},{% endif %}{% endfor %}
一直以来，如果不配置其中的memcached_servers的的话，每个服务每个子进程都会创建一个字典作为cache存放从keystone获得的token，但是字典内容都是相似的，如果使用统一的memcache，就不会有这方面的问题。现在master版本keystonemiddleware认为这是不合适的，会导致不一致的结果和较高的内存占用，计划在N或者O版本移除这个特性。
补充：存储horizon用户会话数据
这个是django支持的功能，这里不作讨论，有兴趣的可以看The Django Book 2.0--中文版

cache相关的具体的配置项可以参考kolla项目中的cache配置，应该还是准确合适的。