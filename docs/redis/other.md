# <center>缓存血崩、穿透、击穿
## 血崩
由于缓存系统扛不住高峰期流量，导致大量的流量流向DB，把DB打死，继而影响整个业务。

即使重启了DB也不能有效解决问题，很快又被打死了。

解决方案:

* 缓存处使用集群部署，以增加读取性能。
* 缓存处使用redis cluster，并需要开启持久化，即使重启后也有部分缓存数据。
* 在应用系统中对请求进行限流处理。

## 穿透
由于请求的某个KEY不在缓存中，进而会去DB中查。正常情况来说这个操作是没问题的。
但是如果黑客恶意发送了大量的无效的KEY，这样就会使DB的压力大增。

解决方案:
> 把无效的KEY在应用层如果能过滤就过滤掉。不能过滤的，就把这些KEY缓存起来。

## 击穿
由于某个KEY的访问量特别大，它又有过期时间，当在某一个时刻过期时，大量的请求也会到达DB。
给DB带来压力。

解决方案:
> 在访问数据库之前加锁处理，只有拿到锁的线程才可以访问DB。然后再更新缓存。


## 缓存和数据库的一致性
这个先更新哪个都不合适。都会出现一个更新成功，另一个更新失败的情况。
方案:

* 先更新数据库。然后把要删除的缓存信息放入消息队列中，再消息该消息，最后
  删除缓存即可。
* 延迟双删：即先删除缓存，再修改数据库，最后再发一个定时任务来删除缓存。