# <center> Index
## Clustered index
每张表的数据都被划分为页(pages)，组成表的这些页被存储在一个叫作`B-tree`索引的数据结构中。表的中数据和辅助索引(secondary indexes)都使用了这样的结构。代表了一整张
表的`B-tree`索引也被称为聚集索引`clustered index`,它是通过主键列`primary key columns`组织起来的。

聚集索引的叶子结点包含了一行`row`中的所有列。并且相邻的叶子结点之间是通过双向链表连接的，这样可以方便实现范围查找。

InnoDB建议为每张表都显示的创建一个主键，如果没有显示的创建的话，那么InnoDB会按以下两种方式来确定主键：
1. 如果没有给表指定一个主键，那么InnoDB会使用表中第一个`UNIQUE`索引中所有被定义为`NOT NULL`的列来创建主键。
2. 如果一个表没有创建主键，也没有一个合适的唯一索引，那么InnoDB会在一个包含`DB_ROW_ID`的合成列上创建一个名为`GEN_CLUST_INDEX`的主键。

## Secondary index
辅助索引的叶子结点只包含索引列和主键列

## Covering index
覆盖索引(Covering Index)是指辅助索引包含了所有的查询字段。直接使用辅助索引中的数据替代了根据辅助索引中的主键再去回表的操作，节省了磁盘IO。

当然覆盖索引技术只能用于在查询表的时候，没有其他事务在更新表，只有在更新事务完成后，再去查询才能使用覆盖索引技术。

## ICP(index condition pushdown)
索引下推ICP(Index Condition Pushdown)是对MySQL使用索引从表中查询数据时的一种优化手段。

在没有ICP时，存储引擎使用索引来定位要查询的数据在表中的位置，并且将这些数据返回给MySQL Server，然后MySQL Server会去执行`WHERE`条件的匹配。

在使用ICP时，如果`WHERE`中部分条件能够被索引中的某些列进行匹配操作，那么MySQL Server会把这部分的`WHERE`条件下推给存储存储引擎,然后存储引擎就会再去匹配这些被下推
的`WHERE`条件，只有匹配成功时，才会把该行返回给MySQL Server以从表中读取完成的数据。

ICP的特点：

* ICP可以减少回表的数据。
* ICP可以用在`InnoDB`和`MyISAM`表，包括这两者的分区表

## 案例
假如在`people`中创建了一个`(zipcode,lastname,firstname)`的辅助索引，
```sql
SELECT * FROM people
  WHERE zipcode='95054'
  AND lastname LIKE '%etrunia%'
  AND address LIKE '%Main Street%';
```
上面这个查询在没有ICP时，只能使用`zipcode='95054'`来查询出需要回表的范围，`lastname LIKE '%etrunia%'`这个虽然是索引的一部分，但是并不能用来限制需要回表的量。所以这个查询就需要从表中把所有`zipcode='95054'`的记录全部查询出来，然后再进行后续`WHERE`条件的匹配。

在启用了ICP后，先使用索引来定位`zipcode='95054'`这部分数据，然后再根据`lastname LIKE '%etrunia%'`再来过滤一次，因为`zipcode`和`lastname`都是辅助索引的一部分，不需要额外的查询，同时又能减少后面需要回表的数据量。这就避免了那些满足`zipcode='95054'`但是不满足`lastname`数据的读取操作。

## 配置
ICP默认是打开的，但是可以通过以下配置来修改
```
SET optimizer_switch = 'index_condition_pushdown=off';
SET optimizer_switch = 'index_condition_pushdown=on';
```
