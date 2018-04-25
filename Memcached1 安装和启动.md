# Memcached的安装和启动

Memcached基于libevent的事件处理，linux系统安装memcached，首先要先安装libevent库：

```bash
# yum install libevent libevent-devel
```

yum安装memcached

```bash
# yum install memcached
```

源码安装memcached

```bash
# wget https://memcached.org/latest
需要重命名文件为memcached-1.x.x.tar.gz
# tar -zxf memcached-1.x.x.tar.gz
# cd memcached-1.x.x
# ./configure --prefix=/usr/local/memcached
# make && make test && sudo make install
```

配置文件

```bash
# cat /etc/sysconfig/memcached
PORT="18888"
USER="memcached"
MAXCONN="1024"
CACHESIZE="64"
OPTIONS="-l 10.10.11.42"
```

## memcached启动

### systemd启动

```shell
# systemctl start memcahced
# systemctl is-active memcached
active
# ps -ef | grep memcached
memcach+  5299     1  0 14:51 ?        00:00:00 /usr/bin/memcached -u memcached -p 18888 -m 64 -c 1024 -l 10.10.11.42
```

### memcached命令启动

```shell
# /usr/bin/memcached -u memcached -p 11211 -m 64 -c 1024 -l 10.10.11.42 -d
# systemctl is-active memcached
unknown
# ps -ef | grep memcac
memcach+  5734     1  0 15:26 ?        00:00:00 /usr/bin/memcached -u memcached -p 11211 -m 64 -c 1024 -l 10.10.11.42 -d
```

参数说明：

```shell
# memcached -h
-h						帮助
-p <num>				设置TCP端口号(默认设置为: 11211)
-U <num>				UDP监听端口(默认: 11211, 0 时关闭) 
-l <ip_addr>			监听地址(默认:所有都允许,无论内外网或者本机更换IP，有安全隐患)
-c <num>				最大同时连接数 (default: 1024)
-d						以daemon方式运行
-u <username>			绑定使用指定用于运行进程<username>
-m <num>				允许最大内存用量，单位M (默认: 64 MB)
-M						禁止LRU策略
-n, --slab-min-size=<bytes> min space used for key+value+flags (default: 48)
-f						增长因子，默认1.25
-P <file>				将PID写入文件<file>，这样可以快速终止进程, 需要与-d 一起使用
-I						slab页的大小，范围1K～128M，默认1M
-v, --verbose			verbose (print errors/warnings while in event loop)
-vv						very verbose (also print client commands/responses)
-vvv					extremely verbose (internal state transitions)
```