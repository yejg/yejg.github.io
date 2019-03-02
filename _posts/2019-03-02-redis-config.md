---
layout: post
title: redis配置详解
categories: [redis]
description: redis配置详解
keywords: redis,sentinel,config
---

### redis配置

```
# Redis配置文件样例

# Note on units: when memory size is needed, it is possible to specify
# it in the usual form of 1k 5GB 4M and so forth:
#
# 1k => 1000 bytes
# 1kb => 1024 bytes
# 1m => 1000000 bytes
# 1mb => 1024*1024 bytes
# 1g => 1000000000 bytes
# 1gb => 1024*1024*1024 bytes
#
# units are case insensitive so 1GB 1Gb 1gB are all the same.

# Redis默认不是以守护进程的方式运行，可以通过该配置项修改，使用yes启用守护进程
# 启用守护进程后，Redis会把pid写到一个pidfile中，在/var/run/redis.pid
daemonize yes

# 当配置[daemonize yes]时，Redis默认会把pid写入pidfile文件，
# 可以通过pidfile指定路径
pidfile /var/run/redis.pid

# 指定Redis监听端口，默认端口为6379
# 如果指定0端口，表示Redis不监听TCP连接
port 6379

# 指定redis 只接收来自于该 IP 地址的请求
# 如果不进行设置，那么将处理所有请求
# bind 127.0.0.1

# 配置unix socket来让redis支持监听本地连接
# unixsocket /tmp/redis.sock

# 配置unix socket使用文件的权限
# unixsocketperm 755

# 当客户端闲置多长时间后关闭连接（0 表示关闭该功能）
timeout 0

# tcp keepalive参数
# 如果设置不为0，就使用配置tcp的SO_KEEPALIVE值，使用keepalive有两个好处:
#   1）检测挂掉的对端。
#   2）降低中间设备出问题而导致网络看似连接却已经与对端断开的问题。
#
# 在Linux内核中，设置了keepalive，redis会定时给对端发送ack。
#   检测到对端关闭，需要两倍的设置值的时间
# 其他内核系统下，则依赖内核配置
# 推荐设置60秒
tcp-keepalive 60

# 指定日志记录级别
# Redis总共支持四个级别：debug、verbose、notice、warning，默认为verbose
#   debug (很多信息, 对开发/测试比较有用)
#   verbose (many rarely useful info, but not a mess like the debug level)
#   notice (moderately verbose, what you want in production probably)
#   warning (only very important / critical messages are logged)
loglevel verbose

# 指定记录日志的文件，默认为标准输出
# 如果配置为redis为守护进程方式运行，而这里又配置为stdout，那么日志将会发送给/dev/null
logfile /var/log/redis.log 

# 是否打开记录syslog功能
# syslog-enabled no

# 指定syslog标识
# syslog-ident redis

# 设置日志的来源设备，必须设置为USER 或 LOCAL0-LOCAL7
# syslog-facility local0

# 设置数据库的数量，默认数据库为0，可以使用select <dbid>命令选择一个db
databases 16

################################ SNAPSHOTTING  #################################
# 指定在多长时间内，有多少次更新操作，就将数据同步到数据文件，可以多个条件配合
# 快照配置（redis数据写到磁盘）:
#
#   save <seconds> <changes>
#
#   Will save the DB if both the given number of seconds and the given
#   number of write operations against the DB occurred.
#
#   下面的3行配置 满足以下条件将会同步数据:
#     900秒（15分钟）内至少1个key值改变（则进行数据库保存--持久化） 
#     300秒（5分钟）内至少10个key值改变（则进行数据库保存--持久化） 
#     60秒（1分钟）内至少10000个key值改变（则进行数据库保存--持久化
#   Note: 可以把所有“save”行注释掉，这样就取消写磁盘操作了
save 900 1
save 300 10
save 60 10000

# 指定存储至本地数据库时是否压缩数据，默认为yes，Redis采用LZF压缩
#   yes：压缩，但是需要一些cpu的消耗
#   no：不压缩，需要更多的磁盘空间
rdbcompression yes

# rdb文件名称，默认值为dump.rdb
dbfilename dump.rdb

# 工作目录
# rdb、aof文件都会写在这个目录
# 注意，这里只能指定一个目录，不能指定文件名
dir ./

################################# REPLICATION #################################
# 主从复制。使用slaveof从 Redis服务器复制一个Redis实例。
# 注意，该配置仅限于当前slave有效
# 设置当本机为slav服务时，设置master服务的ip地址及端口，在Redis启动时，它会自动从master进行数据同步
# slaveof <masterip> <masterport>

# 当master服务设置了密码保护时，slav服务连接master的密码
# masterauth <master-password>

# 当从库同主机失去连接或者复制正在进行，从机库有两种运行方式：
#    1) 如果slave-serve-stale-data设置为yes(默认设置)，从库会继续响应客户端的请求（只不过数据可能不是最新的）。
#    2) 如果slave-serve-stale-data设置为no，除去INFO和SLAVOF命令之外的任何请求都会返回一个错误”SYNC with master in progress”。
slave-serve-stale-data yes

# slave根据指定的时间间隔向服务器发送ping请求，默认时间10秒
# repl-ping-slave-period 10

# 复制连接超时时间。master和slave都有超时时间的设置。
# master检测到slave上次发送的时间超过repl-timeout，即认为slave离线，清除该slave信息。
# slave检测到上次和master交互的时间超过repl-timeout，则认为master离线。
# 需要注意的是repl-timeout需要设置一个比repl-ping-slave-period更大的值，不然会经常检测到超时
# repl-timeout 60

################################## SECURITY ###################################
# 设置Redis连接密码，如果配置了连接密码，客户端在连接Redis时需要通过auth <password>命令提供密码，默认关闭
# requirepass foobared

# 把危险的命令给修改成其他名称。
# 比如CONFIG命令可以重命名为一个很难被猜到的命令，这样用户不能使用，而内部工具还能接着使用
# Example:
#
# rename-command CONFIG b840fc02d524045429941cc15f59e41cb7be6c52
#
# rename-command CONFIG ""

################################### LIMITS ####################################
# 设置同一时间最大客户端连接数，默认10000，Redis可以同时打开的客户端连接数为Redis进程可以打开的最大文件描述符数，
# 当客户端连接数到达限制时，Redis会关闭新的连接并向客户端返回max Number of clients reached错误信息
# maxclients 10000

# 指定Redis最大内存限制，Redis在启动时会把数据加载到内存中，达到最大内存后，Redis会先尝试清除已到期或即将到期的Key，
# 当此方法处理后，仍然到达最大内存设置，将无法再进行写入操作，但仍然可以进行读取操作。
# maxmemory <bytes>

# 内存容量超过maxmemory后的处理策略。
#  volatile-lru：利用LRU算法移除设置过过期时间的key。
#  volatile-random：随机移除设置过过期时间的key。
#  volatile-ttl：移除即将过期的key，根据最近过期时间来删除（辅以TTL）
#  allkeys-lru：利用LRU算法移除任何key。
#  allkeys-random：随机移除任何key。
#  noeviction：不移除任何key，只是返回一个写错误。
# 
# Note: 上面的这些移除策略，如果redis没有合适的key驱逐，对于写命令，还是会返回错误。
#       redis将不再接收写请求，只接收get请求。
#       写命令包括: set setnx setex append
#       incr decr rpush lpush rpushx lpushx linsert lset rpoplpush sadd
#       sinter sinterstore sunion sunionstore sdiff sdiffstore zadd zincrby
#       zunionstore zinterstore hset hsetnx hmset hincrby incrby decrby
#       getset mset msetnx exec sort
#
# The default is:
#
# maxmemory-policy volatile-lru

# lru检测的样本数。
# 使用lru或者ttl淘汰算法，从需要淘汰的列表中随机选择sample个key，选出闲置时间最长的key移除。
# maxmemory-samples 3

############################## APPEND ONLY MODE ###############################
# 默认redis使用的是rdb方式持久化，这种方式在许多应用中已经足够用了。
# 但是redis如果中途宕机，会导致可能有几分钟的数据丢失，根据save来策略进行持久化
# Append Only File是另一种持久化方式，可以提供更好的持久化特性。
# Redis会把每次写入的数据在接收后都写入appendonly.aof 文件，每次启动时Redis都会先把这个文件的数据读入内存里，先忽略RDB文件。
appendonly no

# aof文件名，默认为appendonly.aof
# appendfilename appendonly.aof

# 指定更新日志条件，共有3个可选值：
# no:表示等操作系统进行数据缓存同步到磁盘（快）
# always:表示每次更新操作后手动调用fsync()将数据写到磁盘（慢，安全）
# everysec:表示每秒同步一次（折衷，默认值）
appendfsync everysec
# appendfsync no
# appendfsync always

# 在aof写文件的时候，会执行大量IO
# 此时对于everysec和always的aof模式来说，执行fsync会造成阻塞过长时间
# redis有bgrewriteaof机制，在一个子进程中进行aof的重写，从而不阻塞主进程对其余命令的处理
# 不过问题在于：同时在执行bgrewriteaof操作和主进程写aof文件的操作，两者都会操作磁盘，而bgrewriteaof往往会涉及大量磁盘操作，
#    同时在执行bgrewriteaof操作和主进程写aof文件的操作，两者都会操作磁盘，而bgrewriteaof往往会涉及大量磁盘操作，
#    这样就会造成主进程在写aof文件的时候出现阻塞的情形，现在no-appendfsync-on-rewrite参数出场了。
#     设置为no，是最安全的方式，不会丢失数据，但是要忍受阻塞的问题
#     设置为yes，这时候并没有执行磁盘操作，只是写入了缓冲区，因此这样并不会造成阻塞，但是如果这个时候redis挂掉，就会丢失数据。在linux的操作系统的默认设置下，最多会丢失30s的数据。
no-appendfsync-on-rewrite no

# aof自动重写配置。
# 当目前aof文件大小超过上一次重写的aof文件大小的百分之多少进行重写，
# 即当aof文件增长到一定大小的时候Redis能够调用bgrewriteaof对日志文件进行重写。
# 当前AOF文件大小是上次日志重写得到AOF文件大小的二倍（设置为100）时，自动启动新的日志重写过程。
auto-aof-rewrite-percentage 100
# 设置允许重写的最小aof文件大小，避免了达到约定百分比但尺寸仍然很小的情况还要重写
auto-aof-rewrite-min-size 64mb

################################ LUA SCRIPTING  ###############################
# redis执行lua脚本最大时间限制（毫秒）
# 达到最大时间时，redis会记个log，然后返回error。
#   此时，只有SCRIPT KILL和SHUTDOWN NOSAVE可以用。第一个可以杀没有调write命令的东西。要是已经调用了write，只能用第二个命令杀。
#
# 设置为0则表示不限制
lua-time-limit 5000

################################## SLOW LOG ###################################
# slog log是用来记录redis运行中执行比较慢的命令耗时。
# 当命令的执行超过了指定时间，就记录在slow log中，slog log保存在内存中，所以没有IO操作。
# 执行时间比slowlog-log-slower-than大的请求记录到slowlog里面，单位是微秒，所以1000000就是1秒。
# 注意，负数时间会禁用慢查询日志，而0则会强制记录所有命令。
slowlog-log-slower-than 10000

# 慢查询日志长度。当一个新的命令被写进日志的时候，最老的那个记录会被删掉。理论上这个长度没有限制，只要有足够的内存就行。
# 你可以通过 SLOWLOG RESET 来清楚slowlog释放内存。
slowlog-max-len 128

############################### ADVANCED CONFIG ###############################
# hash类型的数据结构在编码上可以使用ziplist和hashtable
# ziplist的特点就是文件存储(以及内存存储)所需的空间较小,在内容较小时,性能和hashtable几乎一样
# 因此redis对hash类型默认采取ziplist。如果hash中条目的条目个数或者value长度达到阀值,将会被重构为hashtable
hash-max-ziplist-entries 512
hash-max-ziplist-value 64

# Similarly to hashes, small lists are also encoded in a special way in order
# to save a lot of space. The special representation is only used when
# you are under the following limits:
list-max-ziplist-entries 512
list-max-ziplist-value 64

# Sets have a special encoding in just one case: when a set is composed
# of just strings that happens to be integers in radix 10 in the range
# of 64 bit signed integers.
# The following configuration setting sets the limit in the size of the
# set in order to use this special memory saving encoding.
set-max-intset-entries 512

# Similarly to hashes and lists, sorted sets are also specially encoded in
# order to save a lot of space. This encoding is only used when the length and
# elements of a sorted set are below the following limits:
zset-max-ziplist-entries 128
zset-max-ziplist-value 64

# Active rehashing uses 1 millisecond every 100 milliseconds of CPU time in
# order to help rehashing the main Redis hash table (the one mapping top-level
# keys to values). The hash table implementation Redis uses (see dict.c)
# performs a lazy rehashing: the more operation you run into an hash table
# that is rehashing, the more rehashing "steps" are performed, so if the
# server is idle the rehashing is never complete and some more memory is used
# by the hash table.
# 
# The default is to use this millisecond 10 times every second in order to
# active rehashing the main dictionaries, freeing memory when possible.
#
# If unsure:
# use "activerehashing no" if you have hard latency requirements and it is
# not a good thing in your environment that Redis can reply form time to time
# to queries with 2 milliseconds delay.
# 指定是否激活重置哈希，默认为开启
activerehashing yes

# 客户端buffer控制
# 在客户端与server进行的交互中,每个连接都会与一个buffer关联,此buffer用来队列化等待被client接受的响应信息
# 如果client不能及时的消费响应信息,那么buffer将会被不断积压而给server带来内存压力.如果buffer中积压的数据达到阀值,将会导致连接被关闭,buffer被移除
# buffer控制类型<class>包括:
#   normal -> 普通连接
#	slave -> 与slave之间的连接
#	pubsub -> pub/sub类型连接，此类型的连接往往会产生此种问题，因为pub端会密集的发布消息,但是sub端可能消费不足
#
# 指令格式:client-output-buffer-limit <class> <hard limit> <soft limit> <soft seconds>
#   其中：hard表示buffer最大值,一旦达到阀值将立即关闭连接;
#         soft表示"容忍值",它和seconds配合,如果buffer值超过soft且持续时间达到了seconds,也将立即关闭连接,
#		 如果超过了soft但是在seconds之后，buffer数据小于了soft,连接将会被保留.
# 如果hard和soft都设置为0,则表示禁用buffer控制.通常hard值大于soft
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60

# Redis server执行后台任务的频率,默认为10，建议采用默认值。
# 此值越大,表示redis对"间歇性task"的执行次数越频繁(次数/秒)。
# "间歇性task"包括 过期集合检测、关闭空闲超时的连接等，此值必须大于0且小于500。
# 此值过小就意味着更多的cpu周期消耗,后台task被轮询的次数更频繁。
# 此值过大意味着"内存敏感"性较差。
hz 10

################################## INCLUDES ###################################
# 指定包含其他的配置文件，可以在同一主机上多个Redis实例之间使用同一份配置文件，而同时各实例又拥有自己的特定配置文件
# include /path/to/local.conf
# include /path/to/other.conf
```

### sentinel配置

```
# Example sentinel.conf

# port <sentinel-port>
# 哨兵sentinel实例运行的端口，默认26379
port 26379

# sentinel monitor <master-name> <ip> <redis-port> <quorum>
# 哨兵sentinel监控的redis主节点的 ip port   
#   master-name 可以自己命名的主节点名字 只能由字母A-z、数字0-9 、这三个字符".-_"组成
#   quorum 当这些quorum个数sentinel哨兵认为master主节点失联，那么这时客观上就认为主节点失联了  
sentinel monitor masterName 192.18.24.146 6379 2

# sentinel auth-pass <master-name> <password>
# 当在Redis master实例中开启了requirepass foobared，这里就需要设置密码
sentinel auth-pass masterName 123456

# sentinel down-after-milliseconds <master-name> <milliseconds>
# 指定多少毫秒之后，主节点没有应答哨兵sentinel，此时，哨兵主观上认为主节点下线，默认30秒  
sentinel down-after-milliseconds masterName 30000

# sentinel can-failover <master-name> <yes|no>
# 指定此sentinel是否可以为此主服务器启动故障转移
sentinel can-failover masterName yes

# sentinel parallel-syncs <master-name> <numslaves>
# 这个配置项指定了在发生failover主备切换时最多可以有多少个slave同时对新的master进行 同步，  
# 这个数字越小，完成failover所需的时间就越长  
# 但是如果这个数字越大，就意味着越 多的slave因为replication而不可用。  
# 可以通过将这个值设为 1 来保证每次只有一个slave 处于不能处理命令请求的状态。
sentinel parallel-syncs masterName 1

# sentinel failover-timeout <master-name> <milliseconds>
#
# 故障转移超时时间，如果在该时间（ms）内未能完成failover操作，则认为该failover失败
# 默认15 minutes.
sentinel failover-timeout masterName 900000

# 脚本配置
#  
# 配置当某一事件发生时所需要执行的脚本，可以通过脚本来通知管理员，例如当系统运行不正常时发邮件通知相关人员。  
# 对于脚本的运行结果有以下规则：  
#   若脚本执行后返回1，那么该脚本稍后将会被再次执行，重复次数目前默认为10  
#   若脚本执行后返回2，或者比2更高的一个返回值，脚本将不会重复执行。  
#   如果脚本在执行过程中由于收到系统中断信号被终止了，则同返回值为1时的行为相同。  
#   一个脚本的最大执行时间为60s，如果超过这个时间，脚本将会被一个SIGKILL信号终止，之后重新执行。  
#
# 通知型脚本:
#    当sentinel有任何警告级别的事件发生时（比如说redis实例的主观失效和客观失效等等），将会去调用这个脚本[需确保脚本存在且可执行]  
#    这时这个脚本应该通过邮件，SMS等方式去通知系统管理员关于系统不正常运行的信息。
#      调用该脚本时，将传给脚本两个参数：  
#        事件的类型  
#        事件的描述  
#    Sample：  
#      sentinel notification-script <master-name> <script-path>  
#      sentinel notification-script mymaster /var/redis/notify.sh  
#
#  
# 客户端重新配置主节点参数脚本：  
#    当一个master由于failover而发生改变时，这个脚本将会被调用，通知相关的客户端关于master地址已经发生改变的信息。  
#    以下参数将会在调用脚本时传给脚本:  
#       <master-name> <role> <state> <from-ip> <from-port> <to-ip> <to-port>
#         <state> is "start", "end" or "abort"
#		  <role> is either "leader" or "observer"
#		   
#    Sample：  
#      sentinel client-reconfig-script <master-name> <script-path>  
#      sentinel client-reconfig-script mymaster /var/redis/reconfig.sh  
```

