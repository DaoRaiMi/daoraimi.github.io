# <center>主从复制
## Redis主从复制机制：
* 当master和slave之间的连接正常时，master持续的把自己接收到的命令发送到slave端，这样slave端就可以实现和master端相同的数据集了

* 如果master和slave之间的连接断开了(比如网络问题，或master，slave任意一端超时了), 那么slave端会去重新向master发起连接，
并且尝试进行部分同步(partial resynchronization)，这意味着，slave端只会从master获取在网络不可用这段时间内，slave端错过的新的命令。

* 当部分同步不可用时，slave端就会向master请求一次完全同步。这将会导致master进行一次RDB快照，然后再将快照发送给slave, 最后再持续地把收到的命令发送给slave端。

Redis复制默认使用异步同步方式，这种方式具有低延迟、高性能的优势。异步同步的方式能够满足大多数的redis使用场景。

slave端会异步对master进行已接收命令的确认，所以master不必等待slave每个命令的处理，然则master如果想知道slave执行到哪了，执行了什么命令，它还是可以知道的。

在master节点没有开启持久化时，master节点不能自动重启，因为没有持久化，在重启后会导致slave节点
上的数据也会被清空。

## 主从复制原理
　　每个master节点都有一个replication-id，它是一个很大的伪随机string串，是指定数据集的标识。每个master节点还携带的有发往slave
节点的replication-stream的偏移量，是为了让slave节点能够感知数据集新的更新。即使master上没有连接slave节点，这个Offset也会增加的。
```
Replication ID, Offset
```
1. 当slave节点连接master节点时，slave节点使用PSYNC命令向master节点告知它们旧的master的replication id, 以及目前处理到的Offset。
```shell
psync REPLICATION-ID OFFSET
```
2. 这样master节点就可以只发送增量的部分给slave节点。然而，如果master的缓冲区中没有足够的backlog时，或者请求的是一个历史的replication-id
时，那么就会发生全量的同步。
  > 全量同步过程：  
  * master在后台开启一个子进程，用来生成RDB文件，同时它还会缓存从客户端发来的写命令。  
  * 当RDB文件生成好以后，master会把这个RDB文件传输到slave节点上，slave节点先把它保存在自己的磁盘上，然后再加载到内存中。
  * master再把缓存的命令发送到slave上，这样slave就能和master保持一致了。
  * 如果master同时收到了多个全量同步的请求，则master只会生成一次RDB文件，当文件就绪后直接将文件发送到所有的slave上。

## 什么是Replication ID
如果两个节点拥有相同的replication id和replication offset，则说明这两个节点拥有相同的数据集。所以很有必要搞清楚什么是replication id,
以及为什么实际上实例拥有两个replication id: main id 和 secondary id

每当实例从头重启并且作为master时，或slave节点被提升为master节点时，Redis都会为这个节点重新生成一个replication id，当slave和
master节点握手后，slave节点会继承master的replication id，所以两个拥有相同replication id的实例，它们拥有相同的数据集，但是
有可能是不同时期的数据。所以replication offset就充当了逻辑时间的角色。

例如：如果A，B两个实例拥有相同的replication id, 但是A的Offset是1000, B的Offset是1024，那就说明A实例的数据集中少应用了
某些命令，也就说A要再应用几个命令才能达到B实例的状态。

实例需要有两个replication id的原因就是：**slave被提升为了master**
> 在故障转移后，被提升为master角色的那台slave节点仍然保留了它原来的replication id, 因为这个replication id是以前的master的
> 
> 这样，其他的slave节点再和该节点(master)同步时，就会尝试使用旧replication id进行一次部分同步。
> 
> 这样是能够按预期生效的，因为当slave节点被提升为master节点后，它会把它的secondary id设置成它的main id, 并且记录下Offset。
> 然后该节点会再生成一个新的随机的replication id，并把这个新的ID作为它的main id，因为开启了一个新的历史。
> 
> 当在该节点(master)再处理新的复制连接时，master会将它们的replication id，offset与main id, secondary id进行匹配。

简单地说：就是当发生故障转移后，slave再和新的master节点进行同步的时候，可以不必进行全量同步。

至于为什么在发生故障转移后，新的master节点需要重新生成一个replication id，其实也比较简单，比如说在网络发生分区的时候，如果不重新
生成replication id, 那么这就违反了 相同的replication id, offset，拥有相同的数据集这个定义。因为网络发生分区后，这两个的
数据集肯定是不一样的。

## Redis-slave是如何处理过期的Key的？
1. slave节点不进行key的过期操作，相反它们等待master节点来处理过期的key，当master过期了一个key后，它会给所有的slave同步一个DEL操作来删除对应的已经过期了的key。
2. 由于是通过master来删除过期的key的，所以当master节点还未把DEL命令发送过来之前，slave节点上是会存在实际上已经过期了的KEY。为了解决这个问题，slave节点使用自己的逻辑时钟(logical clock)对于那些不会引起数据不一致的读操作，直接返回访问的key不存在。通过这种方式，slave就避免了把在逻辑上已经过期了的key返回给客户端了。
3. 在lua-scripts执行期间是不会有key过期的，因为从概念上来讲，当lua-script运行期间，master节点上的计时器是冻结的。这样做就是为了保证在lua-script执行期间某些key是一直存在或不存在的。这也就保证了在lua-script运行期间不会有key过期，也能保证相同的lua脚本能够在slave节点得到相同的运行结果。

一旦某个slave节点被提升为了master，那么它就可以立即进行过期处理，并不需要从旧的master那里得到任何帮助。
