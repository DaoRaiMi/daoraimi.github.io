# <center>InnoDB Multi Version

InnoDB是一个多版本的存储引擎。它通过保存被修改行的旧版本，进而支持像`concurrency`和`rollback`这样的事务特性的操作。这些信息是以`rollback segment`数据结构存储在`undo tablespace`中。

InnoDB使用`undo tablespace`中的`rollback segment`里的信息来执行事务中的`rollback`操作。

在进行一致性读的时候，InnoDB也使用这些信息来构建被读取行的上一个版本。

在内部，InnoDB给数据库中每一行都添加了以下三个字段：

* `DB_TRX_ID`(6-byte)
  > 表示上一个插入或更新该行的事务的唯一标识。同样，删除操作在内部来说也是一个更新操作，只不过是设置该行的一个特殊位，来标识当前行被删除了。
* `DB_ROLL_PTR`(7-byte)
  > 这个字段也被称为回滚指针`roll pointer`。回滚指针指向的是`rollback segment`中的一个`undo log`。如果该行被更新了，那么`undo log`就包含了重建该行未被更新前的数据所需要的必要信息。
* `DB_ROW_ID`(6-byte)
  > 包含一个行ID，当新行插入时，`DB_ROW_ID`会单调递增。如果InnoDB自动地生成了聚集索引(clustered index)，那么聚集索引中会包含`DB_ROW_ID`的值，否则`DB_ROW_ID`列不会出现在任何索引中。

  
  `rollback segment`中的`undo log`可以分为`insert undo log`和`update undo log`。

  * `insert undo log`只在事务回滚的时候需要，只要事务提交了，`insert undo log`就能够被忽略。
  * `update undo log`也被用在一致性读，但是`update undo log`只有在没有事务使用它的时候才可以被忽略。
