# Memcached监控

### Memcached日志

默认情况下，memcached并不输出日志，可以利用“-v”参数将memcached的信息定向到日志文件中。

```shell
# /usr/bin/memcached -d -u memcached -p 18888 -m 64 -c 1024 -l 10.10.11.42 -vv >> /var/log/memcached.log 2>&1
# /usr/bin/memcached -p 11211 -u memcached -m 1588 -c 8192 -l 0.0.0.0 -U 11211 -t 8 >> /var/log/memcached.log 2>&1
```



### Memcached 图形界面

利用php-memcache实现：

https://pecl.php.net/package/memcache

```shell
# yum install httpd php
# wget https://pecl.php.net/get/memcache-2.2.7.tgz
# tar -zxvf memcache-2.2.7.tgz
# cp memcache-2.2.7/memcache.php /var/www/html/
编辑 memcache.php
# systemctl start httpd
```

浏览器访问 memcached_server/memcache.php，输入账户名和密码

phpMemcachedAdmin

https://github.com/elijaa/phpmemcachedadmin



### Zabbix监控Memcached
