# redis

几种数据结构：
1、string：set、get、del、setnx、setex、setbit、getbit
2、list：lpush、lpop、lrange、lset、lrem
3、hash：hset、hget、hdel、hgetall、mhget、mhset、
4、set：sadd、srem、scard、smembers、sdiff、sunion、sinter
5、zset：zadd、zrem、zcard、zrange


数据结构实现：
每一个key和value都是一个redisObject对象，其中有一个type表示类型，void* ptr指向具体数据结构，key都是string对象，ptr都指向一个某种数据结构的对象。

1、string对象：在value为整数时用int实现，短字符串用embstr（只读）实现，长字符串用raw实现（两者都是sds），float也用sds实现，运算时先转换为float再转换为sds，在需要时会进行编码类型转换，如append把int转换为sds。
2、list对象：在节点长度小于512和字符串长度小于64时，用ziplist实现，不满足条件时用linkedlist实现。
3、hash对象：ziplist或者hashtable，ziplist的kv是相邻的两个节点，当键值对的数量超过512或者字符串长度超过64时，会用把ziplist转换为hashtable。
4、set对象：所有元素都是整数并且数量少于512时，用intset实现，否则用hashtable实现
5、zset对象：用ziplist或者skiplist和hashtable实现，当元素个数大于128或者字符串长度大于64时，转换编码改用skiplist和hashtable，skiplist顺序保存每个元素的名字和分值，hashtable保存名字到分值的映射，以便于查找socre


服务实现：
一个redisServer有16个redisDb对象，client可以通过select 0～15命令来切换db，每个redisDb对象中包含一个dict字典，其中的hashtable的key对象就是用户看到的数据库键，是字符串类型，实现用的是sds，value就是用户看到的数据库值，value对象可以有多种，包括string、list、zset、set、hash这几种，实现的数据结构各不相同

过期机制：
redisDb有一个过期dict，存放每个key和其的过期unix时间戳（如果key被设置了过期，用expire或者setex）
过期删除策略：
1、定期删除：每隔一段时间检查键是否需要删除，serverCron函数执行时会检查，每个数据库顺序检查0~15，每个数据库随机删除n个过期键，如果超出时间限制提前返回。
2、惰性删除：访问到了，并且判断过期，才会删除，可能导致内存长期无法释放。
redis采用定期和惰性删除两种策略结合

对数据库的操作：
select、flushdb、dbsize

持久化：
1、rdb文件：save（阻塞服务）或者bgsave（fork子进程）命令生成，可以设置服务器bgsave执行周期（xxx时间内有xxx次修改），启动时若rdb文件存在则自动载入恢复内存（有aof优先使用aof恢复内存）。
2、aof文件：将每次执行的命令记录下来，通过appendfsync配置文件生成的节奏always（每次都写入磁盘）、everysec（一秒之内有更新就写入磁盘）或者no（系统决定何时写入磁盘），启动时载入文件重新执行所有命令来恢复内存。由于aof文件是通过记录命令来恢复内存，文件会越来越大，redis会通过重写来缩小文件，重写是直接分析内存生成新的aof文件，不需要对原来的aof文件进行处理。

redis服务器：
1、redis是文件事件和时间事件驱动服务，根据当前系统不同采用select、epoll、evport、kqueue等不同io多路复用接口进行编译，若在linux系统下则采用epoll，macos采用kqueue。
2、时间事件通过设置超时来触发，如设置select的超时时间为最近一个需要触发的事件事件，目前redis的时间事件只有一个ServerCron，每隔100ms触发一次，aof文件和过期删除机制就是由该时间事件处理。
3、redis服务器接受请求后顺序、原子的一个个执行，通过dict中的hashtable查找命令对应的处理函数，检查参数是否合法，传入参数进行处理。

主从复制：
1、从服务器执行slaveof ip：port，主服务器执行bgsave生成rdb文件，将rdb文件同步到从服务器，同步完成后再将生成文件期间执行的命令同步到从服务器保证主从一致性。
2、主服务器每次执行的命令都会发送到从服务器，保证主从一致性。
3、从服务器掉线后会先尝试直接同步掉线期间的命令，若主服务器的命令缓冲区已经刷新，则进行完整的同步（第一步的情况）

redis集群：
1、一共有16384个slot，可以用 cluster addslots手动分配，cluster delslots删除，cluster flushslots删除所有slot,cluster keyslots计算key分配到哪个槽。
2、redis-cli --cluster fix ip：port自动分配槽。
3、slot是一个char slot[2048]的数组，一共有2048x8=16384个bit，bit为1表示这个节点拥有这个slot，key如果算到了这个slot上，会把k和v都存到这个redis节点。
4、客户端接收命令时，先计算出key属于哪个slot，再判断这个slot是否属于自己，属于自己则执行命令，否则就产生move错误指引客户端重定向到负责该slot的节点执行命令。
5、一个slot会包含多个key，redis可以迁移slot，迁移时如果客户端执行命令，当前节点没有查到这个key时，会发ask错误到客户端指引客户端重定向到slot迁移的目标节点上进行查找。
6、可以为集群master节点设置从节点，从节点用于复制master节点，master节点下线后，选举该master的一个slave节点接管master节点的任务，当master节点上线后，从节点又会变成slave。
