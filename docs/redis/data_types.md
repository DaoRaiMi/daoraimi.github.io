# <center>Redis基本数据类型
## String
  * string 是 redis 最基本的类型，你可以理解成与 Memcached 一模一样的类型，一个 key 对应一个 value.
  * string 类型是二进制安全的。意思是 redis 的 string 可以包含任何数据。比如jpg图片或者序列化的对象.
  * string 类型的值最大能存储 **512MB**
```
  > 127.0.0.1:6379> set username bob
  > OK
  > 127.0.0.1:6379> get username
  > "bob"
  > 127.0.0.1:6379>
```

## Hash(哈唏)
Hash是一个key-value的集合，key, value都是string类型的。每个hash可以存储$2^{32}-1$个键值对(40多亿)
```
> 127.0.0.1:6379> hset myhash username bob age 18
> (integer) 2
> 127.0.0.1:6379> hgetall myhash
> 1) "username"
> 2) "bob"
> 3) "age"
> 4) "18"
> 127.0.0.1:6379> hget myhash username
> "bob"
> 127.0.0.1:6379>
```

## List(列表)
  > Redis 列表是简单的**字符串**列表，按照**插入顺序**排序(通常是linked-list实现)。你可以添加一个元素到列表的头部（左边）或者尾部（右边）
```
> 127.0.0.1:6379> lpush myList one two three
> (integer) 3
> 127.0.0.1:6379> rpop myList
> "one"
> 127.0.0.1:6379> lrange myList 0 -1
> 1) "three"
> 2) "two"
> 127.0.0.1:6379> lrange myList 0 -1
> 1) "three"
> 2) "two"
> 127.0.0.1:6379>
```

## Set(集合)
Redis 的 Set 是 string 类型的无序集合。集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是 O(1)
```
> 127.0.0.1:6379> sadd mySet java php golang python java
> (integer) 4
> 127.0.0.1:6379> smembers mySet
> 1) "python"
> 2) "java"
> 3) "golang"
> 4) "php"
> 127.0.0.1:6379>
```
**注意**: 上面的集合中添加了两次java，但是最终只有一个，这就体现了集合的唯一性质。合集中最大元素个数为$2^{32}-1$(4294967295, 每个集合可存储40多亿个成员)

## ZSet(有序集合)
Redis zset 和 set 一样也是string类型元素的集合,且不允许重复的成员。不同的是每个元素都会关联一个double类型的分数。

redis正是通过分数来为集合中的成员进行从小到大的排序。zset的成员是唯一的,但分数(score)却可以重复。
```
> 127.0.0.1:6379> zadd myZSet 1 one 2 two 3 three 4 two
> (integer) 3
> 127.0.0.1:6379> ZCOUNT myZSet 1 4
> (integer) 3
> 127.0.0.1:6379> ZINCRBY myZSet 1 one
> "2"
> 127.0.0.1:6379> ZRANGEBYSCORE myZSet 0 10
> 1) "one"
> 2) "three"
> 3) "two"
> 127.0.0.1:6379> ZRANGEBYSCORE myZSet 0 10 withscores
> 1) "one"
> 2) "2"
> 3) "three"
> 4) "3"
> 5) "two"
> 6) "4"
> 127.0.0.1:6379>
```