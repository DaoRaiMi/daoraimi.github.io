# <center>Replication
复制(Replication)能够使数据从一台MySQL实例被复制到另一台或更多台数据库实例中。默认情况下`是异步复制`，复制也并不是需要一直连接到复制源的。可以配置成复制所有的数据库、单个数据库、甚至是选中的表。

## MySQL复制的优点
1. 横向扩展。
   > 可以把读请求分摊到其他的实例上，以提高性能。这种情况下，所有的write, update请求都必须在源服务器上进行，而读请求可以可以在一个或多个副本实例上执行。由于源实例主要是用来做写操作的，所以也能提高写的性能。还可以通过增加多个副本实例来增加读取性能。

2. 数据安全性
> 由于复制过程是能够被暂停的，所以可以在不中断复制源的情况下，在副本上做备份操作。

3. 数据分析
> 可以在副本实例上做数据分析，从而不影响复制源的性能。

4. 远距离的数据分发
> 可以在本地复制远程的实例。也体现了数据的安全性这里。

## 复制方式
1. 基于bin-log文件和position
2. 基于GTID(Global Transaction Identifiers)

## 复制线程
在复制中主要有三个线程，一个在master上，另两个在slave中:

* Binlog dump thread
  > 当slave节点连接上master时，master就会开启一下binlog-dump线程，然后该线程把binlog的内容发送给slave节点。

* Replication I/O receiver thread
  > 当在slave节点上执行`start replica`时，slave节点就会启动这个线程。
  >
  > Replicaiton I/O receiver线程会把它接收到的binlog内容，写到本地的日志文件中，这个日志就是relay-log.

* Replication SQL applier thread
  > SQL线程读取relay-log中的内容，并在本实例上应用相关操作，以保证主从数据一致。

## 主从复制延迟分析
从复制的过程可以看出，主从延迟的主要原因是在master上事务是并行执行的，但是在slave中却只有一个线程去应用relay-log，当并发量特别多的时候，延迟就会特别的高。

在mysql-5.6版本中首次提出了多线程复制，不过这个并不是真正意义上的多线程复制，而是基于database的，也就说从database角度来看是多线程的，但是具体到单个的database内部来看，还是单线程复制的。当然这在一个实例上有多个需要复制的库时，还有是可观的效果的。

于是在mysql-5.7提出了redo-log的group-commit, 还有基于GTID的复制。group-commit是为了解决redo-log频繁刷盘带来的性能上的影响，它可以把多个事务的redo-log一次一起写入到磁盘中。所以，基于group-commit官方双提出了真正的多线程复制。

## 多线程复制
对于每个副本来说，可以使用`replica_parallel_workers`来设置SQL线程的数据(在8.0.26之前是用`slave_parallel_workers`)，该变量在8.0.27之前默认值是`0`，从8.0.27开始默认是`4`.

与此同时，副本节点上还会再启一个协调线程，用来管理这些worker线程。

`replica_parallel_type`的默认值是`LOGICAL_CLOCK`,它还有一个值是`database`。

* LOGICAL_CLOCK
  > 属于同一个group-commit的事务，在slave节点上可以并行的执行。
* DATABASE
  > 在不同database上执行的事务，在slave上可以被并行的执行

`replica_preserve_commit_order`可以保证让事务在slave上的提交顺序和master上的一致。该变量在8.0.27之前默认值为`off`, 之后便是`on`。

**注意** 目前`NDB Cluster`并不支持多线程复制。

## 半同步复制
由于MySQL的复制是异步进行的，所以也就有可能导致数据丢失。所以MySQL也以插件的方式来支持`半同步复制`。

半同步复制，就是说MySQL并不会等待所有的slave节点对binlog事件进行确认。默认只是等待其中的一个。
`rpl_semi_sync_source_wait_for_replica_count`

关于半同步复制的配置官方文档中都有，需要的时候可以参考配置就行。