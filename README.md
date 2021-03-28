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
redisDb有一个过期dict，存放每个key和其的过期unix时间戳（如果key被设置了过期）
过期删除策略：
1、定时删除：设置定时器，到点直接删除。
2、定期删除：每隔一段时间检查键是否需要删除。
3、惰性删除：访问到了才会删除。

对数据库的操作：
select、flushdb、dbsize

持久化：
1、rdb文件：save（阻塞服务）或者bgsave（fork子进程）命令生成，可以设置服务器bgsave执行周期（xxx时间内有xxx次修改），启动时若rdb文件存在则自动载入恢复内存（有aof优先使用aof恢复内存）。
2、aof文件：将每次执行的命令记录下来，通过appendfsync配置文件生成的节奏always（每次都写入磁盘）、everysec（一秒之内有更新就写入磁盘）或者no（系统决定何时写入磁盘），启动时载入文件重新执行所有命令来恢复内存。由于aof文件是通过记录命令来恢复内存，文件会越来越大，redis会通过重写来缩小文件，重写直接分析内存，不需要对原来的aof文件进行处理。

对数据库的操作：
对数据库的操作：
