

# Memcached命令


https://github.com/memcached/memcached/wiki/Protocols

Memcached有三种命令：

​	存储命令

​	查询命令

​	统计命令

#### 连接memcached

```shell
# telnet <host> <port>

# telnet 10.10.11.42 18888
Trying 10.10.11.42...
Connected to 10.10.11.42.
Escape character is '^]'.
stats
STAT pid 6018
STAT uptime 917
……
quit  #退出
```

```shell
[root@wangwenzhi-team-172-28-14-43 ~]# echo "stats settings" | nc 10.10.11.42 18888
STAT maxbytes 67108864
STAT maxconns 1024
STAT tcpport 18888
STAT udpport 18888
STAT inter 10.10.11.42
……
STAT item_size_max 1048576
STAT maxconns_fast no
STAT hashpower_init 0
STAT slab_reassign no
STAT slab_automove 0
END
```



#### Memcached 存储 命令

语法：

```
<command name> <key> <flags> <exptime> <bytes> [noreply]\r\n
<data block>
```

```
command：set/add/replace/append/prepend/CAS
key：key，用于查找缓存值。
flags：一个16-bit的标志值，存储关于键值对的额外信息 。
exptime：在缓存中保存键值对的时间长度（以秒为单位，0 表示永远）
bytes：在缓存中存储的字节数,必须与世纪存储值的长度一致
noreply（可选）： 该参数告知服务器不需要返回数据
data block：存储的值（始终位于第二行）（可直接理解为key-value结构中的value）
```

##### set

set 命令将 **value(数据值)** 存储在指定的 **key(键)** 中。

如果key不存在，则创建新的key-value，如果key已经存在，则更新value。

```
set key1 0 600 1
a
STORED							# 保存成功
get key1
VALUE key1 0 1
a
END
set key1 0 100 1
b
STORED
get key1
VALUE key1 0 1
b
END
set key2 0 600 2
a
CLIENT_ERROR bad data chunk
ERROR							# 保存失败
```

##### add

add 命令创建新的key-value。如果key已经存在，不会更新原来的value值。

```
set key1 0 100 10
key1_value
STORED							# 保存成功
get key1
VALUE key1 0 10
key1_value
END
add key2 0 100 10
key2_value
STORED
get key2
VALUE key2 0 10
key2_value
END
add key1 0 100 8
key1_new
NOT_STORED						# 未保存
get key1
VALUE key1 0 10
key1_value
END

```

##### replace

replace命令替换已存在key的value值

```
add key1 0 100 10
key1_value
STORED
get key1
VALUE key1 0 10
key1_value
END
replace key1 1 200 12
key1_replace
STORED
get key1
VALUE key1 1 12
key1_replace
END
```

##### append

append命令向已存在的key的value值后面追加数据

```
add key1 0 100 10
key1_value
STORED
append key1 0 100 11
key1_append
STORED
get key1
VALUE key1 0 21
key1_valuekey1_append			# 追加的数据在原来的数据后面
END
```

##### prepend

prepend命令向已存在的key的value值前面追加数据

```
set key1 0 600 10 
key1_value
STORED
get key1
VALUE key1 0 10
key1_value
END
prepend key1 0 600 12
key1_prepend
STORED
get key1
VALUE key1 0 22
key1_prependkey1_value			# 追加在原数据前面
END
```

##### CAS

cas指check and set，对已经存在的key-value进行检查并操作。检查是通过<cas unique>实现。

<cas unique> 是一个唯一的 64-bit value，通过gets命令获得。

cas语法格式如下： 

```
cas <key> <flags> <exptime> <bytes> <cas unique> [noreply]\r\n
value
```

```
add key1 0 600 10
key1_value
STORED
gets key1
VALUE key1 0 10 8			# <cas unique> 为 8
key1_value
END
cas key1 1 700 8 8
key1_cas
STORED
gets key1
VALUE key1 1 8 9
key1_cas
END
```

#### Memcached 查找命令

语法：

```
<command name> <key> [noreplay]\r\n
```

```
command name : get/gets/delete/incr/decr
key：key，用于查找缓存值
```

##### get

get命令获取key对应的value值

```
get key1
VALUE key1 0 10
key1_value
END
get key1 key2 key3			# 多个key用空格分开
VALUE key1 0 10
key1_value
VALUE key2 0 10
key2_value
VALUE key3 0 10
key3_value
END
```
```
get 4f992435ab909f54e4fee521ea779aa05b2485b5
VALUE 4f992435ab909f54e4fee521ea779aa05b2485b5 1 431
cdogpile.cache.api
CachedValue
p1
(S'{"instance_uuid": "b9616454-adb5-4103-a5c2-1d68b5d5c70b", "access_url": "http://36.111.0.100:10016/vnc_auto.html?token=ff44f816-b4e6-49b5-a2bb-930452882920", "token": "ff44f816-b4e6-49b5-a2bb-930452882920", "last_activity_at": 1510731606.609711, "internal_access_path": null, "console_type": "novnc", "host": "10.131.73.27", "port": "5900"}'
p2
(dp3
S'v'
I1
sS'ct'
p4
F1510731606.609839
stRp5
.
END
```

##### gets

gets命令获取key对应的value值，并带有cas令牌

```
gets key1
VALUE key1 0 10 13
key1_value
END
gets key1 key2 key3
VALUE key1 0 10 13
key1_value
VALUE key2 0 10 14
key2_value
VALUE key3 0 10 15
key3_value
END
```



```
gets 8507935135891bc3f99b96547ac2d8117c77c0c0
VALUE 8507935135891bc3f99b96547ac2d8117c77c0c0 1 431 205268
cdogpile.cache.api
CachedValue
p1
(S'{"instance_uuid": "cddaa8f8-e0dc-4aad-b704-c2237f0e3fa9", "access_url": "http://36.111.0.100:10016/vnc_auto.html?token=b94cd298-b20f-4a47-af73-5800fd09da32", "token": "b94cd298-b20f-4a47-af73-5800fd09da32", "last_activity_at": 1512396185.241457, "internal_access_path": null, "console_type": "novnc", "host": "10.131.73.26", "port": "5916"}'
p2
(dp3
S'v'
I1
sS'ct'
p4
F1512396185.241586
stRp5
.
END

```



#####delete

delete命令删除key-value

```
get key1 key2 key3
VALUE key1 0 10
key1_value
VALUE key2 0 10
key2_value
VALUE key3 0 10
key3_value
END
delete key1
DELETED
delete key2
DELETED
delete key3
DELETED
get key1
END
```

#####incr/decr

incr和decr对已存在的value值进行增减操作，只对十进制整数有效

```
add key1 0 600 2
10
STORED
get key1
VALUE key1 0 2
10
END
incr key1 2
12
get key1
VALUE key1 0 2
12
END
decr key1 3
9
get key1
VALUE key1 0 1
9
END
```



##### Touch

The "touch" command is used to update the expiration time of an existing item
without fetching it.

touch <key> <exptime> [noreply]\r\n

##### Get And Touch

The "gat" and "gats" commands are used to fetch items and update the
expiration time of an existing items.

gat <exptime> <key>*\r\n
gats <exptime> <key>*\r\n

- <exptime> is expiration time.
- <key>* means one or more key strings separated by whitespace.





#### Memcached统计命令

语法：

```
stats [items | slabs | sizes | settings]\r\n
```

##### stats

stats输出Memcached的通用统计信息，输出格式如下：

```
STAT <name> <value>\r\n
```

```
stats
STAT pid 10409
STAT uptime 9308
STAT time 1515660932
STAT version 1.4.15
……
STAT total_items 19
STAT expired_unfetched 0
STAT evicted_unfetched 0
STAT evictions 0
STAT reclaimed 2
END
```

```
|-----------------------+---------+-------------------------------------------|
| Name                  | Type    | Meaning                                   |
|-----------------------+---------+-------------------------------------------|
| pid                   | 32u     | Process id of this server process         |
| uptime                | 32u     | Number of secs since the server started   |
| time                  | 32u     | current UNIX time according to the server |
| version               | string  | Version string of this server             |
| pointer_size          | 32      | Default size of pointers on the host OS   |
|                       |         | (generally 32 or 64)                      |
| rusage_user           | 32u.32u | Accumulated user time for this process    |
|                       |         | (seconds:microseconds)                    |
| rusage_system         | 32u.32u | Accumulated system time for this process  |
|                       |         | (seconds:microseconds)                    |
| curr_items            | 64u     | Current number of items stored            |
| total_items           | 64u     | Total number of items stored since        |
|                       |         | the server started                        |
| bytes                 | 64u     | Current number of bytes used              |
|                       |         | to store items                            |
| max_connections       | 32u     | Max number of simultaneous connections    |
| curr_connections      | 32u     | Number of open connections                |
| total_connections     | 32u     | Total number of connections opened since  |
|                       |         | the server started running                |
| rejected_connections  | 64u     | Conns rejected in maxconns_fast mode      |
| connection_structures | 32u     | Number of connection structures allocated |
|                       |         | by the server                             |
| reserved_fds          | 32u     | Number of misc fds used internally        |
| cmd_get               | 64u     | Cumulative number of retrieval reqs       |
| cmd_set               | 64u     | Cumulative number of storage reqs         |
| cmd_flush             | 64u     | Cumulative number of flush reqs           |
| cmd_touch             | 64u     | Cumulative number of touch reqs           |
| get_hits              | 64u     | Number of keys that have been requested   |
|                       |         | and found present                         |
| get_misses            | 64u     | Number of items that have been requested  |
|                       |         | and not found                             |
| get_expired           | 64u     | Number of items that have been requested  |
|                       |         | but had already expired.                  |
| get_flushed           | 64u     | Number of items that have been requested  |
|                       |         | but have been flushed via flush_all       |
| delete_misses         | 64u     | Number of deletions reqs for missing keys |
| delete_hits           | 64u     | Number of deletion reqs resulting in      |
|                       |         | an item being removed.                    |
| incr_misses           | 64u     | Number of incr reqs against missing keys. |
| incr_hits             | 64u     | Number of successful incr reqs.           |
| decr_misses           | 64u     | Number of decr reqs against missing keys. |
| decr_hits             | 64u     | Number of successful decr reqs.           |
| cas_misses            | 64u     | Number of CAS reqs against missing keys.  |
| cas_hits              | 64u     | Number of successful CAS reqs.            |
| cas_badval            | 64u     | Number of CAS reqs for which a key was    |
|                       |         | found, but the CAS value did not match.   |
| touch_hits            | 64u     | Number of keys that have been touched     |
|                       |         | with a new expiration time                |
| touch_misses          | 64u     | Number of items that have been touched    |
|                       |         | and not found                             |
| auth_cmds             | 64u     | Number of authentication commands         |
|                       |         | handled, success or failure.              |
| auth_errors           | 64u     | Number of failed authentications.         |
| idle_kicks            | 64u     | Number of connections closed due to       |
|                       |         | reaching their idle timeout.              |
| evictions             | 64u     | Number of valid items removed from cache  |
|                       |         | to free memory for new items              |
| reclaimed             | 64u     | Number of times an entry was stored using |
|                       |         | memory from an expired entry              |
| bytes_read            | 64u     | Total number of bytes read by this server |
|                       |         | from network                              |
| bytes_written         | 64u     | Total number of bytes sent by this server |
|                       |         | to network                                |
| limit_maxbytes        | size_t  | Number of bytes this server is allowed to |
|                       |         | use for storage.                          |
| accepting_conns       | bool    | Whether or not server is accepting conns  |
| listen_disabled_num   | 64u     | Number of times server has stopped        |
|                       |         | accepting new connections (maxconns).     |
| time_in_listen_disabled_us                                                  |
|                       | 64u     | Number of microseconds in maxconns.       |
| threads               | 32u     | Number of worker threads requested.       |
|                       |         | (see doc/threads.txt)                     |
| conn_yields           | 64u     | Number of times any connection yielded to |
|                       |         | another due to hitting the -R limit.      |
| hash_power_level      | 32u     | Current size multiplier for hash table    |
| hash_bytes            | 64u     | Bytes currently used by hash tables       |
| hash_is_expanding     | bool    | Indicates if the hash table is being      |
|                       |         | grown to a new size                       |
| expired_unfetched     | 64u     | Items pulled from LRU that were never     |
|                       |         | touched by get/incr/append/etc before     |
|                       |         | expiring                                  |
| evicted_unfetched     | 64u     | Items evicted from LRU that were never    |
|                       |         | touched by get/incr/append/etc.           |
| evicted_active        | 64u     | Items evicted from LRU that had been hit  |
|                       |         | recently but did not jump to top of LRU   |
| slab_reassign_running | bool    | If a slab page is being moved             |
| slabs_moved           | 64u     | Total slab pages moved                    |
| crawler_reclaimed     | 64u     | Total items freed by LRU Crawler          |
| crawler_items_checked | 64u     | Total items examined by LRU Crawler       |
| lrutail_reflocked     | 64u     | Times LRU tail was found with active ref. |
|                       |         | Items can be evicted to avoid OOM errors. |
| moves_to_cold         | 64u     | Items moved from HOT/WARM to COLD LRU's   |
| moves_to_warm         | 64u     | Items moved from COLD to WARM LRU         |
| moves_within_lru      | 64u     | Items reshuffled within HOT or WARM LRU's |
| direct_reclaims       | 64u     | Times worker threads had to directly      |
|                       |         | reclaim or evict items.                   |
| lru_crawler_starts    | 64u     | Times an LRU crawler was started          |
| lru_maintainer_juggles                                                      |
|                       | 64u     | Number of times the LRU bg thread woke up |
| slab_global_page_pool | 32u     | Slab pages returned to global pool for    |
|                       |         | reassignment to other slab classes.       |
| slab_reassign_rescues | 64u     | Items rescued from eviction in page move  |
| slab_reassign_evictions_nomem                                               |
|                       | 64u     | Valid items evicted during a page move    |
|                       |         | (due to no free memory in slab)           |
| slab_reassign_chunk_rescues                                                 |
|                       | 64u     | Individual sections of an item rescued    |
|                       |         | during a page move.                       |
| slab_reassign_inline_reclaim                                                |
|                       | 64u     | Internal stat counter for when the page   |
|                       |         | mover clears memory from the chunk        |
|                       |         | freelist when it wasn't expecting to.     |
| slab_reassign_busy_items                                                    |
|                       | 64u     | Items busy during page move, requiring a  |
|                       |         | retry before page can be moved.           |
| slab_reassign_busy_deletes                                                  |
|                       | 64u     | Items busy during page move, requiring    |
|                       |         | deletion before page can be moved.        |
| log_worker_dropped    | 64u     | Logs a worker never wrote due to full buf |
| log_worker_written    | 64u     | Logs written by a worker, to be picked up |
| log_watcher_skipped   | 64u     | Logs not sent to slow watchers.           |
| log_watcher_sent      | 64u     | Logs written to watchers.                 |
|-----------------------+---------+-------------------------------------------|
```

##### stats reset

清空统计数据

##### stats items

stats items 输出每个slab class 的item存储信息，输出格式如下：

```
STAT items:<slabclass>:<stat> <value>\r\n
```

```
stats items
STAT items:4:number 2
STAT items:4:age 4232
STAT items:4:evicted 0
STAT items:4:evicted_nonzero 0
……
END
```

```
Name                   Meaning
------------------------------
number                 Number of items presently stored in this class. Expired
                       items are not automatically excluded.
number_hot             Number of items presently stored in the HOT LRU.
number_warm            Number of items presently stored in the WARM LRU.
number_cold            Number of items presently stored in the COLD LRU.
number_temp            Number of items presently stored in the TEMPORARY LRU.
age_hot                Age of the oldest item in HOT LRU.
age_warm               Age of the oldest item in WARM LRU.
age                    Age of the oldest item in the LRU.
evicted                Number of times an item had to be evicted from the LRU
                       before it expired.
evicted_nonzero        Number of times an item which had an explicit expire
                       time set had to be evicted from the LRU before it
                       expired.
evicted_time           Seconds since the last access for the most recent item
                       evicted from this class. Use this to judge how
                       recently active your evicted data is.
outofmemory            Number of times the underlying slab class was unable to
                       store a new item. This means you are running with -M or
                       an eviction failed.
tailrepairs            Number of times we self-healed a slab with a refcount
                       leak. If this counter is increasing a lot, please
                       report your situation to the developers.
reclaimed              Number of times an entry was stored using memory from
                       an expired entry.
expired_unfetched      Number of expired items reclaimed from the LRU which
                       were never touched after being set.
evicted_unfetched      Number of valid items evicted from the LRU which were
                       never touched after being set.
evicted_active         Number of valid items evicted from the LRU which were
                       recently touched but were evicted before being moved to
                       the top of the LRU again.
crawler_reclaimed      Number of items freed by the LRU Crawler.
lrutail_reflocked      Number of items found to be refcount locked in the
                       LRU tail.
moves_to_cold          Number of items moved from HOT or WARM into COLD.
moves_to_warm          Number of items moved from COLD to WARM.
moves_within_lru       Number of times active items were bumped within
                       HOT or WARM.
direct_reclaims        Number of times worker threads had to directly pull LRU
                       tails to find memory for a new item.
hits_to_hot
hits_to_warm
hits_to_cold
hits_to_temp           Number of get_hits to each sub-LRU.

Note this will only display information about slabs which exist, so an empty
cache will return an empty set.
```

##### stats slabs

"stats slabs" 输出memcached启动后创建的slabs的信息，输出格式如下：

```
STAT <slabclass>:<stat> <value>\r\n
STAT <stat> <value>\r\n
```

```
|-----------------+----------------------------------------------------------|
| Name            | Meaning                                                  |
|-----------------+----------------------------------------------------------|
| chunk_size      | The amount of space each chunk uses. One item will use   |
|                 | one chunk of the appropriate size.                       |
| chunks_per_page | How many chunks exist within one page. A page by         |
|                 | default is less than or equal to one megabyte in size.   |
|                 | Slabs are allocated by page, then broken into chunks.    |
| total_pages     | Total number of pages allocated to the slab class.       |
| total_chunks    | Total number of chunks allocated to the slab class.      |
| get_hits        | Total number of get requests serviced by this class.     |
| cmd_set         | Total number of set requests storing data in this class. |
| delete_hits     | Total number of successful deletes from this class.      |
| incr_hits       | Total number of incrs modifying this class.              |
| decr_hits       | Total number of decrs modifying this class.              |
| cas_hits        | Total number of CAS commands modifying this class.       |
| cas_badval      | Total number of CAS commands that failed to modify a     |
|                 | value due to a bad CAS id.                               |
| touch_hits      | Total number of touches serviced by this class.          |
| used_chunks     | How many chunks have been allocated to items.            |
| free_chunks     | Chunks not yet allocated to items, or freed via delete.  |
| free_chunks_end | Number of free chunks at the end of the last allocated   |
|                 | page.                                                    |
| mem_requested   | Number of bytes requested to be stored in this slab[*].  |
| active_slabs    | Total number of slab classes allocated.                  |
| total_malloced  | Total amount of memory allocated to slab pages.          |
|-----------------+----------------------------------------------------------|
```

##### stats cachedump

stats cachedump输出指定slab的item的key列表，语法格式如下：

    stats cachedump <slab> <limit>\r\n
    <slab>:指定slab
    <limit>:指定显示数量


##### stats sizes

"stats sizes" 输出缓存中所有items的大小和数量
警告: 1.4.27版本 this command causes the cache server to
lock while it iterates the items. 1.4.27 and greater are safe.

输出格式:

```
STAT <size> <count>\r\n
```

```
stats sizes_enable 		# enable the histogram at runtime
stats sizes_disable 	# disable the histogram at runtime
```

#####stats conns

"stats conns" 返回当前的连接信息

```
STAT <file descriptor>:<stat> <value>\r\n
```

```
The following "stat" keywords may be present:

|---------------------+------------------------------------------------------|
| Name                | Meaning                                              |
|---------------------+------------------------------------------------------|
| addr                | The address of the remote side. For listening        |
|                     | sockets this is the listen address. Note that some   |
|                     | socket types (such as UNIX-domain) don't have        |
|                     | meaningful remote addresses.                         |
| state               | The current state of the connection. See below.      |
| secs_since_last_cmd | The number of seconds since the most recently        |
|                     | issued command on the connection. This measures      |
|                     | the time since the start of the command, so if       |
|                     | "state" indicates a command is currently executing,  |
|                     | this will be the number of seconds the current       |
|                     | command has been running.                            |
|---------------------+------------------------------------------------------|

The value of the "state" stat may be one of the following:

|----------------+-----------------------------------------------------------|
| Name           | Meaning                                                   |
|----------------+-----------------------------------------------------------|
| conn_closing   | Shutting down the connection.                             |
| conn_listening | Listening for new connections or a new UDP request.       |
| conn_mwrite    | Writing a complex response, e.g., to a "get" command.     |
| conn_new_cmd   | Connection is being prepared to accept a new command.     |
| conn_nread     | Reading extended data, typically for a command such as    |
|                | "set" or "put".                                           |
| conn_parse_cmd | The server has received a command and is in the middle    |
|                | of parsing it or executing it.                            |
| conn_read      | Reading newly-arrived command data.                       |
| conn_swallow   | Discarding excess input, e.g., after an error has         |
|                | occurred.                                                 |
| conn_waiting   | A partial command has been received and the server is     |
|                | waiting for the rest of it to arrive (note the difference |
|                | between this and conn_nread).                             |
| conn_write     | Writing a simple response (anything that doesn't involve  |
|                | sending back multiple lines of response data).            |
|----------------+-----------------------------------------------------------|
```



##### stats settings

"stats settings" 返回memcached的详细设置

```
|-------------------+----------+----------------------------------------------|
| Name              | Type     | Meaning                                      |
|-------------------+----------+----------------------------------------------|
| maxbytes          | size_t   | Maximum number of bytes allowed in the cache |
| maxconns          | 32       | Maximum number of clients allowed.           |
| tcpport           | 32       | TCP listen port.                             |
| udpport           | 32       | UDP listen port.                             |
| inter             | string   | Listen interface.                            |
| verbosity         | 32       | 0 = none, 1 = some, 2 = lots                 |
| oldest            | 32u      | Age of the oldest honored object.            |
| evictions         | on/off   | When off, LRU evictions are disabled.        |
| domain_socket     | string   | Path to the domain socket (if any).          |
| umask             | 32 (oct) | umask for the creation of the domain socket. |
| growth_factor     | float    | Chunk size growth factor.                    |
| chunk_size        | 32       | Minimum space allocated for key+value+flags. |
| num_threads       | 32       | Number of threads (including dispatch).      |
| stat_key_prefix   | char     | Stats prefix separator character.            |
| detail_enabled    | bool     | If yes, stats detail is enabled.             |
| reqs_per_event    | 32       | Max num IO ops processed within an event.    |
| cas_enabled       | bool     | When no, CAS is not enabled for this server. |
| tcp_backlog       | 32       | TCP listen backlog.                          |
| auth_enabled_sasl | yes/no   | SASL auth requested and enabled.             |
| item_size_max     | size_t   | maximum item size                            |
| maxconns_fast     | bool     | If fast disconnects are enabled              |
| hashpower_init    | 32       | Starting size multiplier for hash table      |
| slab_reassign     | bool     | Whether slab page reassignment is allowed    |
| slab_automove     | bool     | Whether slab page automover is enabled       |
| slab_automove_ratio                                                         |
|                   | float    | Ratio limit between young/old slab classes   |
| slab_automove_window                                                        |
|                   | 32u      | Internal algo tunable for automove           |
| slab_chunk_max    | 32       | Max slab class size (avoid unless necessary) |
| hash_algorithm    | char     | Hash table algorithm in use                  |
| lru_crawler       | bool     | Whether the LRU crawler is enabled           |
| lru_crawler_sleep | 32       | Microseconds to sleep between LRU crawls     |
| lru_crawler_tocrawl                                                         |
|                   | 32u      | Max items to crawl per slab per run          |
| lru_maintainer_thread                                                       |
|                   | bool     | Split LRU mode and background threads        |
| hot_lru_pct       | 32       | Pct of slab memory reserved for HOT LRU      |
| warm_lru_pct      | 32       | Pct of slab memory reserved for WARM LRU     |
| hot_max_factor    | float    | Set idle age of HOT LRU to COLD age * this   |
| warm_max_factor   | float    | Set idle age of WARM LRU to COLD age * this  |
| temp_lru          | bool     | If yes, items < temporary_ttl use TEMP_LRU   |
| temporary_ttl     | 32u      | Items with TTL < this are marked  temporary  |
| idle_time         | 0        | Drop connections that are idle this many     |
|                   |          | seconds (0 disables)                         |
| watcher_logbuf_size                                                         |
|                   | 32u      | Size of internal (not socket) write buffer   |
|                   |          | per active watcher connected.                |
| worker_logbuf_size| 32u      | Size of internal per-worker-thread buffer    |
|                   |          | which the background thread reads from.      |
| track_sizes       | bool     | If yes, a "stats sizes" histogram is being   |
|                   |          | dynamically tracked.                         |
| inline_ascii_response                                                       |
|                   | bool     | If yes, stores numbers from VALUE response   |
|                   |          | inside an item, using up to 24 bytes.        |
|                   |          | Small slowdown for ASCII get, faster sets.   |
|-------------------+----------+----------------------------------------------|
```



##### flush_all

清理缓存中的所有 key-value

#### memcached-tool

memcached-tool是用perl语言写的一个对memcache的状态性能分析分具。

```
Usage: memcached-tool <host[:port] | /path/to/socket> [mode]
    memcached-tool 10.0.0.5:11211 display    # shows slabs
    memcached-tool 10.0.0.5:11211            # same. (default is display)
    memcached-tool 10.0.0.5:11211 stats      # shows general stats
    memcached-tool 10.0.0.5:11211 settings   # shows settings stats
    memcached-tool 10.0.0.5:11211 sizes      # shows sizes stats
    memcached-tool 10.0.0.5:11211 dump       # dumps keys and values
WARNING! sizes is a development command.
As of 1.4 it is still the only command which will lock your memcached instance for some time.
If you have many millions of stored items, it can become unresponsive for several minutes.
Run this at your own risk. It is roadmapped to either make this feature optional
or at least speed it up.
```

```
[root@10e131e73e127 ~]# memcached-tool 10.131.73.127:18888
  #  Item_Size  Max_age   Pages   Count   Full?  Evicted Evict_Time OOM
  3     152B         0s       1       0     yes        0        0    0
  4     192B    594273s       1       1     yes        0        0    0
  5     240B    594273s       1     995     yes        0        0    0
  6     304B    594273s       1       6     yes        0        0    0
  7     384B         0s       1       0     yes        0        0    0
  8     480B         0s       1       0     yes        0        0    0
  9     600B    594273s       1    1432     yes        0        0    0
 10     752B         0s       1       0     yes        0        0    0
 11     944B         0s       1       0     yes        0        0    0
 12     1.2K    594292s       1       4     yes        0        0    0
 13     1.4K         0s       1       0     yes        0        0    0
 14     1.8K         0s       1       0     yes        0        0    0
 15     2.3K         0s       1       0     yes        0        0    0
 19     5.5K       294s       2      26     yes        0        0    0
 21     8.7K    594292s       3     299     yes        0        0    0
 22    10.8K    594288s       7     579     yes        0        0    0
 23    13.6K         0s       1       0     yes        0        0    0
```

