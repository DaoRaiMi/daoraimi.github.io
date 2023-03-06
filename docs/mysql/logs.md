# <center>日志
## bin-log
二进制日志记录那些能够引起数据库发生改变的**事件**，bin-log存在的目的有两个：

1. 用于MySQL的主从复制
2. 用于即时点的还原操作

bin-log记录格式：

* Statement
  > 日志中记录的是SQL语句，有些语句在主从复制场景下是不安全的，也就是说相同的语句在主节点和从节点上会产生不同的结果。比如使用了now(), 随机函数等。所以主从复制场景中，bin-log必须使用ROW格式记录。
* Row
  > 日志中记录的是每一行变更前后的值
* Mix
  > 将statement和row两种格式结合起来使用。默认先使用statement来记录，然后在某些情况下会自动的切换成row格式的来记录。

## redo-log
redo-log其实就是通常说的“事务日志”，默认存储在ib_logfile0, ib_logfile1中，这两个文件是滚动使用的。

在MySQL中事务是WAL(Write Ahead Log，提前写日志)的，所以当出现了未完成的事务，或是已完成的事务，但是没有提交，那么在MYSQL再次启动时会分析redo-log，对于可以提交的事务进行提交处理，不可提交的事务就执行回滚操作。

**相关配置**

* innodb_log_file_size
  > 指定单个redo-log文件的大小
* innodb_log_files_in_group
  > 指定redo-log组中有多少个日志文件，默认是2个
* innodb_flush_log_at_trx_commit:
    * 0: 每秒把redo-log写到文件，并flush-disk，可能会丢失1秒内的数据。但是性能*最高
    * 1: 每当事务提交的时候，就把redo-log写到文件，并进行flush-disk操作。默认操作,安全性最高，不会有数据丢失。但是性能在这三个中是最低的。
    * 2: 每当事务提交的时候wite redo-log，然后每秒进行flush-disk操作。性能和安全性在0和1之间。

**注意**

有很多操作系统和磁盘硬件会“欺骗”我们的flush-to-disk这个操作，它们会给mysqld说数据已经存盘了，即使它们并没有真正的存盘。

所以在这种情况下，即使使用了上面推荐的设置，在极端情况下也不能保证数据的安全性。通常的做法是使用带有电池的磁盘或RAID卡，即使在断电的情况下，也能有足够的时间来把磁盘缓存中的数据真正存盘。

## undo-log
**undo log**包含了如何去恢复某个事务对**cluster index**的修改的信息。如果另一个事务需要查看未修改之前的数据(一致读操作),那么这个原始的数据就会从**undo log**中来获取。

**undo log**存储在**undo log segment**中，而**undo log segment**又在**rollback segments**中。**Rollback segments**是在**undo tablespaces**和全局临时tablespace中。

每个undo表空间和全局临时表空间最多支持128个回滚段。innodb_rollback_segments变量定义了回滚段的数量。
