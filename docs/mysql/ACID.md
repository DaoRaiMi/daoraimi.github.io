# <center>ACID
## 简介
`ACID`是`Atomicity(原子性)、Consistency(一致性)、Isolation(隔离性)和Durability(持久性)`这四个单词首字母的缩写。这四个特性是数据库系统所期望的，并且都与`transaction(事务)`紧密相联。

* Atomicity
  > 原子性是说事务中的操作是一个整体，不可再分，它们要么被`commit`，要么被`rollback`;
* Consistency
  > 在任何时间数据库都保持在一个一致的状态(例如，在每次事务commit或rollback之后、在事务进行之中)，如果一个查询涉及了多个表，那么该查询要么是看到全部的旧数据，要么就是看到全部的新数据，不可能看到一个中间状态的数据。
* Isolation
  > 事务在执行的时候彼此是被隔离起来，事务之间不会相互影响，也不会看到对方未提交的数据。隔离是通过锁来实现的。
* Durability
  > 事务的结果是持久化的，一旦事务提交之后，相应的数据就是安全的，不会受断电，操作系统crash等的影响。

## 事务的隔离级别
MySQL的InnoDB支持事务，支持的事务隔离级别如下：

* READ UNCOMMITTED
  > 读未提交。读取了其他事务还未提交的数据，产生了**脏读**问题

* READ COMMITTED
  > 读已提交。在当前事务中可以读取其他事务已经提交的数据，产生了**不可重复读**问题

* REPEATABLE READ
  > 可重复读。通过MVCC解决不可重复读的问题，通过间隙锁解决了**幻读**问题
  
* SERIALIZABLE
  > 串行事务。

默认的隔离级别是`REPEATABLE READ`