# <center>基本操作
## 数据库管理
```sql
-- 创建数据库
> create database test default charset utf8mb4;

-- 删除数据库
> drop database test;

-- 显示服务器上所有的库
> show databases;

-- 切换数据库
> use test;
```

## 表管理
```sql
-- 创建表
create table student(
    id bigint unsigned auto_increment primary key comment'主键', 
    username varchar(40) not null default '' comment '帐号', 
    nickname varchar(100) not null default '' comment '妮称', 
    age smallint unsigned not null default 0 comment '年龄', 
    address varchar(255) not null default '' comemnt '家庭住址',
    created_at timestamp(3) default current_timestamp(3) not null comment '创建时间', 
    updated_at timestamp(3) not null default current_timestamp(3) \
            on update current_timestamp(3) comment '更新时间', 
    deleted_at timestamp(3) default null comment '删除时间', 
    unique index `uk_username` (`username`)
)engine=innodb;

-- 查看表
> show tables;

-- 查看表结构
> desc student;

-- 查看建表语句
> show create table student;

-- 表重命名
> rename table test to tmp_test

-- 在指定位置增加列名
> alter table student add password varchar(255) default '' not null after username;

> alter table student add email varchar(200) default '' not null after password;

> alter table student add grade_id tinyint unsigned not null default 0 after email, \
add class_id smallint unsigned not null default 0 after grade_id;

-- 重命名列名
> alter table tmp_test change email email_address varchar(100);

-- 修改列类型和约束
> alter table test modify column username varchar(40) not null default '';

-- 增加唯一性约束
> alter table student add constraint unique `uk_email`(`email`);

-- 增加联合索引
> alter table student add index `idx_grade_class_id` (`grade_id`,`class_id`);

-- 删除索引
> alter table student drop index idx_grade_class_id;
> alter table student drop index uk_email;

-- 删除列名
> alter table student drop column class_id, drop column grade_id;
```

## 用户管理
```sql
-- 查看用户列表
> select User,Host from mysql.user;

-- 创建用户
> create user test@localhost identifited by 'abc-123';

-- 给用户授权
> grant all privileges on test.* to test@localhost;

-- 查看用户权限
> show grants for test@localhost;

-- 撤销授权
> revoke all privileges on test.* from test@localhost;

-- 删除用户
> drop user test@localhost;
```