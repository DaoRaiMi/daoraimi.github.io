# <center>Redis持久化
## 持久化方案：
1. RDS(Redis Database)
   > RDB持久化是周期性的对redis的数据集做即时的快照来实现的。
2. AOF(append only file)
   > AOF持久化会记录服务器接收到的每个写操作，当服务器启动的时候会再次的被重放，这样就能重建redis的数据集了。这此命令会以redis protocol格式来记录，当AOF文件变得很大时，redis会在后台对AOF进行重写操作。
3. No Persistence: 用户可以完全关闭redis的数据持久化的功能。
   
4. RDB + AOF
   > 可以在redis中同时开启rdb和aof两种持久化方案。但是需要注意的是，在这种情况下，如果redis重启后，那么会使用AOF来重建数据集，因为AOF中有更加完整的数据。

## RDB和AOF的特点
### RDB优点
* RDB是一个非常紧凑的文件，代表了redis的数据集，所以RDB文件非常适合用来做备份。
* RDB文件非常适合用来做故障恢复，由于是紧凑型文件，所以可以被上传到数据中心，或S3
* RDB最大化了redis的性能，因为redis的主进程需要做的仅仅是FORK一个子进程来做，它不会涉及任何和RDB相关的DISK I/O
* 在大数据集情况下，与AOF相比，RDB可以更快的进行重启操作
* 在主从复制时，RDB支持在重启或故障转移后的**部分同步**(partial resynchronization)
### RDB缺点
* 在redis发生故障时，RDB并不能做到丢失数据的最小化。
* RDB使用FORK子进程来进行数据的持久化，但是当数据集比较大时，这个过程可能会非常的耗时，并且还有可能导致redis毫秒级，甚至秒级延迟。
### AOF优点
* AOF拥有更加可能的持久性，可以设置不同的fsync策略：
    * no fsync
      > 依靠OS的定期flush来落盘(Linux默认是30秒)。如果redis服务器故障，可能会丢失OS落盘周期内的数据。
    * 每秒fsync，
      > 这是**默认**的策略，在这种情况下redis的性能也是非常强的。(fsync是通过一个后台线程来执行的，当没有fsync线程在执行时，主线程会尽可能的执行写操作)。
    * 每次执行命令时fsync
      > 不会有数据的丢失，但是性能是三者中最差的。
* AOF日志文件是一个只追回的文件，所以当redis服务器断电时，不会出现数据seek和不完整的问题。即使由于某些原因AOF日志文件出现了写了一半的情况，使用**redis-check-aof**工具也能很快的修复。
* 当AOF文件变大时，redis能够自动地在后台对AOF文件进行重写。日志重写操作是安全的，因为在重写时，redis的写操作还是在向旧的日志文件中写入。重写完成后，会生成一个完整的新的AOF文件，该文件中包含了重建数据集所需要的最少的命令操作。一旦重写完成，redis就会交换新旧AOF文件，redis后续的写操作都会基于新的AOF来写入。
* AOF文件的格式也是非常容易理解的。比如在执行了FLUSHALL命令后，在还没有进行AOF重写时，可以把AOF文件中的FLUSHALL命令删除，然后就可以恢复数据了。
### AOF缺点
* 在同样数据集的情况下，AOF文件通常要比RDB的文件要大
* 在某些fsync策略下，AOF可能会比RDB性能低一点。通常在默认每秒fsync策略下，redis的性能也是非常高的，在不主动fsync时，可以达到和RDB同等的性能。
* AOF在过去使用特定的命令时(BRPOPLPUSH)，在重写后会导致数据不一致的问题。这种BUG是非常罕见的，redis的开发者也进行了大量的测试都是正常的。但是这种BUG在RDB中是不会出现的。

### AOF过程
1. redis通过fork创建一个子进程，现在我们就有了两个进程；
2. 子进程开始进行AOF日志重写操作，结果保存在一个临时文件中；
3. 父进程在AOF重写开始后，就把接收到的命令都写入内存缓冲区，同是也还会向旧的AOF文件中追加，这样能够保证当重写失败时，数据还是完整的；
4. 当子进程完成了重写操作后，父进程会收到信号，然后把内存缓冲区中的日志追加到子进程产生的临时AOF文件中；
5. redis会自动的把临时文件重命名成AOF文件，并且把此后的日志都追加到新的日志文件中。


### 用户该如何选择？
　　官方建议是将RDB和AOF组合起来使用，如果说想提高数据的持久性的话，因为RDB可以提供更快的重启速度，更为方便的备份方式，以及可能出现的AOF引擎的BUG。

## 备份策略
　　RDB文件一旦生成后，就不会再进行修改，所以在redis运行期间进行复制

1. 建议每小时进行一次RDB快照(SAVE, BGSAVE)，保存最近48小时备份；
2. 每天进行一次RDB快照，保存到另一个目录中；
3. 每天必须把备份的文件备份到异地；

　　如果只启用了AOF持久化，则也可以对AOF文件进行周期性的复制。虽然可能会出现最后一部分的命令缺失，但是redis在启动的过程中能够自动修复的。

## 内存淘汰策略
`maxmemory`: 设置一个内存使用的最大值，当内存使用达到最大值后，会根据`maxmemory-policy`的设置去删除相应的key

`maxmemory-policy`

* volatile-lru -> Evict using approximated LRU, only keys with an expire set.
* allkeys-lru -> Evict any key using approximated LRU.
* volatile-lfu -> Evict using approximated LFU, only keys with an expire set.
* allkeys-lfu -> Evict any key using approximated LFU.
* volatile-random -> Remove a random key having an expire set.
* allkeys-random -> Remove a random key, any key.
* volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
* noeviction -> Don't evict anything, just return an error on write operations.

**LRU** means Least Recently Used
   > 最近最久未使用，实现：把数据放入一个链表的头部，如果链表中的数据被命中，则将对应的数据移动到链表的头部，当链表满了，需要再放入新的数据时，则从链表的尾部删除一个数据。

**LFU** means Least Frequently Used
   > 最近最不经常使用：把数据放入到一个链表中，当命中后就把数据的使用频次+1，当链表满后，则把使用频次最小的那个数据删除。

**注意:** 

slave节点上默认会忽略掉maxmemory的设置，因为其数据主要来自于master，而这两个实例的硬件配置也至少是相同的，所以slave可以忽略该配置。但是也可以使用`replica-ignore-maxmemory no`配置项来让maxmemory在slave上生效。