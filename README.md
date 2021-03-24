# redis

几种数据结构：
1、string：set、get、del、setnx、setex、setbit、getbit
2、list：lpush、lpop、lrange、lset、lrem
3、hash：hset、hget、hdel、hgetall、mhget、mhset、
4、set：sadd、srem、scard、smembers、sdiff、sunion、sinter


数据结构实现：
每一个key和value都是一个redisObject对象，其中有一个type表示类型，void* ptr指向具体数据结构，key都是string对象，ptr都指向一个某种数据结构的对象。

1、string对象：在value为整数时用int实现，短字符串用embstr（只读）实现，长字符串用raw实现（两者都是sds），float也用sds实现，运算时先转换为float再转换为sds，在需要时会进行编码类型转换，如append把int转换为sds。
2、list对象：在节点长度小于512和字符串长度小于64时，用ziplist实现，不满足条件时用linkedlist实现。
3、hash对象：ziplist或者hashtable，ziplist的kv是相邻的两个节点，当键值对的数量超过512或者字符串长度超过64时，会用把ziplist转换为hashtable。
4、set对象：所有元素都是整数并且数量少于512时，用intset实现，否则用hashtable实现
