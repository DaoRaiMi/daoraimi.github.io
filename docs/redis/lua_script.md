# <center>Redis-lua script
从redis-2.6版本开始，redis内置了一个lua解释器，我们可以使用**eval**或**evalsha**来执行lua脚本。

```
> eval "return {KEYS[1],KEYS[2],ARGV[1],ARGV[2], ARGV[3]}" 2 key1 key2 first second third
1) "key1"
2) "key2"
3) "first"
4) "second"
5) "third"
```
* 第一个参数：也就是**""**中的内容，其实就是lua 5.1的一个脚本。
  > 注意这个参数是必须的，即使没有KEY，也要传一个0
* 第二个参数：2，这个是key的个数，对应了其后面的key1, key2。可以使用KEYS[1..N]来获取
* key后面的参数就是脚本的参数了。可以使用 ARGV[1...N] 来获取

在lua脚本中可以使用下面两个函数来执行redis命令：
* redis.call()
* redis.pcall()
  它们的唯一区别就在于返回错误的方式不一样。其他的都是一样的。
```
127.0.0.1:6379> eval "return redis.call('get','username')" 0
"Jim"
127.0.0.1:6379> eval "return redis.pcall('set', KEYS[1], ARGV[1])" 1 username bob
OK
127.0.0.1:6379> eval "return redis.call('get','username')" 0
"bob"
127.0.0.1:6379>
```