redis
===
字符串类型
---
* 字符串操作
	+ GET key
	+ SET key value
	+ DEL key
	+ APPEND key value
	+ GETRANGE key start end
	+ SETRANGE key start value
* 整形操作
	+ INCR key
	+ DECR key
	+ INCRBY key amount
	+ DECRBY key amount
* 二进制操作
	+ GETBIT key position
	+ SETBIT key start value
	+ BITCOUNT key [start end]
	+ BITOP AND/OR/XOR/NOT dest-key k1 [k2 k3 ...]
	
列表类型
---
* RPUSH key value
* LPUSH key value
* RPOP key
* LPOP key
* LINDEX key position
* LRANGE key start end
* LTRIM key start end
* BLPOP key timeout
* BRPOP key timeout
* RPOPLPUSH source-key dest-key
* BRPOPLPUSH source-key dest-key

集合
---
* SADD key value
* SREM key value
* SISMEMBER key value
* SCARD key
* SMEMBERS
* SRANDMEMBER key [count]
* SPOP key
* SMOVE source-key dest-key value
* SDIFF key [keys]
* SDIFFSTORE dest-key key [keys]
* SINTER key [keys]
* SINTERSTORE dest-key key [keys]
* SUNION key [keys]
* SUNIONSTORE dest-key key [keys]

散列
---
* HMGET key key
* HMSET key key value
* HDEL key key
* HLEN key
* HEXISTS key key
* HKEYS key
* HVALS key
* HGETALL key
* HINCRBY key key amount
* HINCRBYFLOAT key key amout

有序集合
---
* ZADD key score member
* ZREM key member
* ZCARD key
* ZINCRBY key amount member
* ZCOUNT key min max
* ZRANK key member
* ZSCORE key member
* ZRANGE key start stop
* ZREVRANK key member
* ZREVRANGE key start stop
* ZRANGEBYSCORE key min max
* ZREVRANGEBYSCORE key min max
* ZREMRANGEBYRANK key start stop
* ZREMRANGEBYSCORE key min max
* ZINTERSTORE dest-key key-count key [keys] [SUM/MIN/MAX]
* ZUNIONSTORE dest-key key-count key [keys] [SUM/MIN/MAX]