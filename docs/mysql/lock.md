# <center>InnoDB Locking

## 共享锁(Shared Locks)和排它锁(Exclusive Locks)
InnoDB实现了标准的**行级锁**：`Shared Locks(S)`和`exclusive locks(X)`.
* Shared Lock允许持有该锁的事务对当前行进行读操作。
* Exclusive Lock允许持有该锁的事务对当前行进行更新或删除操作

如果事务`T1`持有行`r`的`S`锁，那么另一个事务`T2`对行`r`的加锁流程如下：
* 如果`T2`申请的是`S`锁，则会立即获得锁，此时事务`T1`和`T2`对都持有行`r`上的`S`锁
* 如果`T2`申请的是`X`锁，则不会立即获得锁。

如果事务`T1`持有的是行`r`上的`X`锁，那么事务`T2`不管申请什么行锁都不会立即获得锁，相反它会等待事务`T1`释放行`r`上的`X`锁。

## 意向锁(Intention Locks)
InnoDB支持多粒度锁(Multi Granularity Locking)，这就使得表锁和行锁能够共存。例如：`LOCK TABLES ... WRITE`会在指定的表上面添加排它锁`X`，为了能够实现多粒度的锁，InnoDB使用了意向锁(intention locks).

意向锁是表级锁，它表示事务在后续会在表中的行上添加哪种类型(Shared or Exclusive)的锁。意向锁会为以下两种：
* Intention Shared Lock(IS)意向共享锁：表示事务想要在表中的某些行上添加`S`锁。
  > `SELECT ... FOR SHARE`就是一个`IS`锁。

* Intention Exclusive Lock(IX)意向排它锁：表示事务想要在表中的某些行上添加`X`锁。
  > `SELECT ... FOR UPDATE`就是一个`IX`锁。

下面是`表级锁`兼容图表：
|   | X | IX | S | IS |
|:-:|:-:|:-: |:-:|:-: |
|X  |冲突|冲突|冲突|冲突|
|IX |冲突|兼容|冲突|兼容|
| S |冲突|冲突|兼容|兼容|
|IS |冲突|兼容|兼容|兼容|

意向锁不会阻塞任何操作除了全表请求(full table requests)例如(`LOCK TABLE ... WRITE`)。意向锁的主要用途是显示哪人操作将要或者正在锁定表中的行。

加在表上的意向锁的信息可以使用`SHOW ENGINE INNODB STATUS`来查看
```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

## 记录锁(Record Locks)
记录锁(Record Locks)是在索引记录上加的锁，即便一个表没有定义任何索引，那么InnoDB也会自动的为表创建一个隐藏的主键，然后再用这个主键实现记录锁。

记录锁的信息可以使用`SHOW ENGINE INNODB STATUS`来查看：
```sql
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t`
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
 2: len 7; hex b60000019d0110; asc        ;;
```

## 间隙锁(Gap Locks)
间隙锁锁住的是索引记录间的间隙，或者在第一个索引记录之前的间隙，又或者最后一个索引记录之后的间隙。

例如`SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE`，阻止了其他事务向把`t.c1 = 15`的值插入到表中，不管这个值是否存在表中，因为这个范围内的所有的值都已经被锁定了。

一个间隙锁可能横跨单个索引值、多个索引值，或者为空。

间隙锁是性能和并发两者之间权衡的一部分，并且在某些特定的事务隔离级别中使用。

通过唯一索引来查询唯一的一行的时候，是不需要用到间隙锁的（当然，如果查询条件中只包含了联合唯一索引中的某些列，此时间隙锁还是会发生）。

例如：
```sql
SELECT * FROM child WHERE id = 100 for update;
```
* 当`id`列上有唯一索引时，上面的查询就只需要`record lock`就行了，不需要间隙锁。
* 如果`id`不是一个唯一索引，那么上面这个查询就会锁定前面的间隙。
* 如果`id`上没有任何索引，那么查询将会锁定整张表，也就是退化为表锁。

需要注意的是不同的事务是可以在相同的间隙中持有相互冲突的锁的。例如：事务A对间隙上持有共享间隙锁(Gap S-lock)，同时事务B可以在相同的间隙上持有一个互斥间隙锁(Gap X-lock)。允许出现这种情况的原因就是如果某个记录从索引中被删除了，那么由不同的事务在该记录上持有的间隙锁就必须要做合并处理。

InnoDB中的间隙锁是“纯禁止的”，也就是说间隙锁的目的就是为了防止其他事务向间隙中插入数据。间隙锁(shared,exclusive)之间是可以共存的。一个事务对某个间隙加上了间隙锁，并不是阻止其他事务对相同的间隙再加上一个间隙锁。Shared 间隙锁和Exclusive间隙锁之间没有区别，它们之间不会相互冲突，都具有相同的功能。


## Next-Key Locks
Next-Key锁是记录锁(Record Lock)和间隙锁(索引记录前面的间隙)的组合。

InnoDB是以这样的方式来加行锁(row-level locking)的，当它查找或扫描一个表的索引时，它在索引记录上设置一个shared或exclusive锁，因此，行锁实际上是索引记录锁(index-record locks)。索引记录上的Next-Key锁也会影响该索引记录前面的间隙，也就是说一个nex-key锁就是一个`索引记录锁(index-record lock)`加上一个索引记录之前间隙的`间隙锁`。如果某个会话在索引中持有了某个记录(R)的shared或exclusive锁，那么其他的会话是不能在记录(R)的前面(按索引序列)间隙中插入一个新的索引记录。

假如有一个索引包含了10,11,13和20这几个值，那么对于Next-Key锁来说，有以下可能的区间：
```
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```
就最后个区间来说，Next-Key锁会锁定最大值以上的间隙以及**supremum**这个伪记录(supremum是在索引中比任何值都要大的一个伪记录)，所以，实际上Next-Key锁锁定的是索引值中最大值后面的间隙。

默认情况下，InnoDB运行在`REPEATABLE READ`隔离级别，在这种情况下，InnoDB使用Next-Key锁来执行索引的查找和扫描，这样的话就可以避免**幻读**的发生。


## 查看加锁情况
```sql
SELECT * FROM PERFORMANCE_SCHEMA.DATA_LOCKS\G
```

