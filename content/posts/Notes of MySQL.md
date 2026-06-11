+++
title = "Notes of MySQL"
date = 2023-07-13

[taxonomies]
tags = ["mysql", "innodb", "database"]
+++



# MySQL

## SQL

### DDL

数据定义语言 Data Definition Language

```mysql
show databases;
select database0;
create database [if not exists] base_name [default charset] [collate];
drop database [if exists] base_name;
use base_name;
#************************#
show tables;
desc table_name;
show create table;
create table table_name(
		field_1 field_type [comment cmt],
  	...
)[comment cmt];
#************************#
alter table table_name modify target_field new_type(len);
alter table table_name change old_field new_field new_type(len) [comment cmt] [restrain];
alter table table_name drop field;
alter table table_name rename to new_table_name;
drop table [if exists] table_name;#删目录
truncate table table_name;#删内容
```

### DML

数据操作语言 Data Manipulation Language

```mysql
insert into table_name(field1,...) values(val1,...);
insert into table_name values(val...);
insert into table_name(field1,...) values(val1,...),...;
insert into table_name values(val...),...;
#************************#
update table_name set field1=val1,... [where condition];
#************************#
delete from table_name [where condition];
```

### DQL

数据查询语言 Data Query Language

- 基本查询
- 条件查询(where)
- 聚合查询(count, max, min, avg, sum)
- 分组查询(group by)
- 排序查询(order by)
- 分页查询(limit)

```mysql
select field1,... from table_name;
select * from table_name;
select field [[as] niknme],... from table_name;
select distinct field from table_name;
#************************#
select ... from table_name where condition;
select ... from table_name in (val1,...);
select ... from table_name between ... and ...;#[]
select ... from table_name like '_?' or '%?';
#************************#
select ... from ... [where con] group by evidence_field [hanving con];
select ... from ... order by field1 asc_desc,...;
#************************#
select ... from ... limit start_index, following_amount;
```

### DCL

数据控制语言 Data Control Language

```mysql
use mysql;
select * from user;
create user 'username'@'host' identified by 'pwd';
alter user 'username'@'host' identified with old_pwd by 'new_one';
drop user 'username'@'host';
#************************#
show grants for 'username'@'host';
grant grant_list on base.table to 'username'@'host';
revoke grant_list on base.table from 'username'@'host';
```

## 函数

字符串函数

- Concat(s1,...)

- Lower(str)
- Upper(str)
- LPad(str,n,pad)
- RPad(str,n,pad)
- Trim(str)
- Substring(str, start,len)

数值函数

- Ceil(x)
- Floor(x)
- Mod(x,y) x/y的模
- Rand()
- Round(x,y) 四舍五入，保留y位小数

日期函数

- Curdate()
- Curtime()
- Now()
- Year(date)
- Month(date)
- Day(date)
- Date_add(date,INTERVAL expr type)
- Datediff(date1,date2)

流程函数

- if(value t, f)
- Ifnull(value1, value2)
- Case when [val1] then [res1] ... else [default] end
- Case [expr] then [res1] ... else [default] end

## 约束

1. 非空 not null
2. 唯一 unique
3. 主键 primary key (AUTO_INCREMENT)
4. 默认 default
5. 检查 check
6. 外键 foreign key

## 多表查询

关系

1. 一对多：多的一方设置外键，关联另一方主键
2. 多对多：建立中间表，中间表包含两个外键，关联两张表主键
3. 一对一：用于表结构拆分，任意表中设置唯一外键，关联另一表主键

查询

1. 内连接

    - 隐式：select ... from A,B where ...
    - 显式：select ... from A inner join b on ...

2. 外连接

    - 左外：select ... from A left join B on ...
    - 右外：select ... from A right join B on ...

3. 自连接

    Select ... from A a1,A a2 where ...

4. 子查询

    标量子查询、列子查询、行子查询、表子查询

## 事务

@@autocommit=0 手动提交事务

commit 提交事务

rollback 回滚

事务的四大特性

- 原子性 Atomicity
- 一致性 Consistency
- 隔离性 Isolation
- 持久性 Durability

事务并发存在的问题

1. 脏读：读另一个事务没有提交的数据
2. 不可重复读：先后读一个记录，但是数据不同（期间有其他事务提交）
3. 幻读：无法插入

隔离级别

1. Read uncommitted
2. Read commited
3. Repeatable Read
4. Serializable

Select @@transaction_isolation

set [session|global] transaction isolation level {level1234}



## 锁

### 全局锁

加锁后，整个数据库实例就处于只读状态，典型场景：全库逻辑备份

`flush tables with read lock;`

`mysqldump -uroot -p密码 basename > base.sql` 终端执行

`unlock tables`

1. 备份期间，所有业务停摆
2. 如果在库上备份，那么在备份期间不能从库执行主库同步过来的二进制日志会导致主从延迟

在InnoDB引擎中，可以在备份时加上参数--single-transaction来完成不加锁的一致性数据备份

### 表级锁

#### 表锁

- 表共享读锁 read lock
- 表独占写锁 write lock

`lock tables name read/write`

`unlock tables `

读锁阻塞写，写锁阻塞所有操作。

#### 元数据锁 meta data lock, MDL

由系统自动控制，无需显式使用

事务A，B开启，操作对象表t

1. A select t
2. B select lock_info

![Screenshot 2023-07-12 at 10.19.00](/images/table_lock_1.png)

3. B insert bob into employee

![image-20230712102406447](/images/table_lock_2.png)

4. A select * from employee

![Screenshot 2023-07-12 at 10.25.28](/images/table_lock_3.png)

5. A insert id3
6. B alter id to primary key (blocked) exclusive与其他不兼容
7. A commit;
8. B throws error, 'cause A insert id3, primary key cannot be add

#### 意向锁

- 意向共享锁 IS `select ... lock in share mode`添加

​		与表锁共享锁兼容（read），与表锁排他锁（write）互斥

- 意向排他锁 IX `insert、update、delete、select ... for update`添加

​		与所有表锁互斥，意向锁之间不排斥

当表已经存在意向锁时试图加互斥表锁，会导致加表锁的事务线程阻塞，直到意向锁解除

### 行级锁

#### 行锁 Record Lock

锁定单行的纪录，防止其他事务update和delete

共享锁 S 排他锁 X 与之前类似![Screenshot 2023-07-12 at 10.51.57](/images/Record_lock.png)

InnoDB使用next-key锁进行搜索和索引扫描，以防止幻读

1. 针对唯一索引进行检索时，对已经存在的纪录进行等值匹配时，将自动优化为行锁
2. InnoDB的行锁是针对于索引加的锁，不通过索引条件检索数据，那么InnoDB将对所有记录加锁，此时升级为**表锁**。（SQL优化update）

#### 间隙锁 GapLock

锁定索引记录的间隙，不包含行记录，确保间隙不变，防止其他事务insert，不同事务的间隙锁可以兼容

InnoDB使用next-key锁扫描，

1. 索引上的等值查询（唯一索引），给不存在的纪录加锁时优化为间隙锁

|  id  | name | age  |
| :--: | :--: | :--: |
|  3   | PHP  |  3   |
|  8   | Java |  4   |

A：`update table set age = 10 where id = 5;`

B：`insert into table values(7,Python,7);` (blocked)![Screenshot 2023-07-14 at 10.59.57](/images/gap_lock.png)

#### 临键锁 Next-Key Lock

行锁和间隙锁的结合，同时锁住数据和间隙

2. 索引上的等值查询（普通索引），遍历到最后一个值不满足查询需求时，next-key lock退化为间隙锁
3. 索引上的范围查询（唯一索引），会访问到不满足的条件的第一个值为止

锁的范围表示，间隙锁范围左开右闭。

## InnoDB引擎

### 逻辑存储结构

![点击查看图片来源](/images/storing_structure.png)

#### 表空间 ibd

一个mysql实例可以对应多个表空间，用于存储记录、索引等数据

#### 段 segment

- Leaf node segment 数据段
- Non-leaf node segment 索引段
- Rollback segment 回滚段

InnoDB是索引组织表，数据段就是B+树的叶子结点，索引段为非叶子结点。段用来管理多个Extent

#### 区 extent

表空间的单元结构，每个区大小为1M。默认下InnoDB存储引擎页大小为16K，即一个区中一共有64个连续的页

#### 页 page

InnoDB存储引擎管理磁盘的最小单元，每页大小默认16KB。为保证页的连续性，InnoDB每次从磁盘申请4-5个区

#### 行 row

InnoDB数据是按行存放的

- Trx_id 每次对某条记录进行改动时，都会把对应的事务id赋值给trx_id隐藏列。
- Roll_pointer 每次对某条记录进行改动时，都会把旧的版本写入到undo日志中，如何隐藏列就相当于一个指针可以用来找到该记录修改前的信息

### 架构

![Screenshot 2023-07-17 at 08.56.06](/images/db_structure.png)

#### 内存结构

- 缓冲池 主内存的一个区域
    - 单位 Page
        - free 空闲未被使用
        - clean 被使用但没修改过
        - dirty 数据被修改过
- 自适应哈希索引
    - 用于优化对缓冲池数据的查询，InnoDB会监控对表上各索引页的查询，如果观察到hash索引可以提升速度，则建立hash索引
- 更改缓冲区 
    - 存储变更的数据，之后再同步到缓冲池，最后刷新到磁盘
- 日志缓冲区
    - 保存要写入到磁盘中的log日志数据（redo log、undo log），默认大小16MB，日志缓冲区的日志回定期刷新到磁盘中。如果需要更新、插入或删除许多行的事务，增加日志缓冲区的大小可以节省磁盘I/O
    - innodb_log_buffer_size 缓冲区大小
    - innodb_flush_log_at_trx_commit 刷新到磁盘时机
        - 1 日志每次事务提交时写入并刷新到磁盘
        - 0 每秒写入并刷新到磁盘一次
        - 2 日志在每次事务提交后写入并每秒刷新到磁盘一次？？？？

#### 磁盘结构

- System Tablespace
    - 系统表空间，更改缓冲区的存储区域
- File-Per-Table Tablespaces 
    - 独立表空间，每个表的文件表空间包含单个InnoDB表的数据和索引，并存储在文件系统上的单个数据文件中
- General Tablespace
    - 通用表空间，需要通过create tablespace创建，在创建表时，可以指定该表空间 e.g. Create tablespace xxx add datafile xxx engine = engine_name
- Undo Tablespaces
    - 撤销表空间，MySQL实例在初始化时会自动创建两个默认的undo表空间（初始大小16M），用于存储undo日志
- Temporary Tablespaces
    - 临时表空间，使用临时表空间和全局临时表空间存储用户创建的临时表 
- Doublewrite Buffer Files
    - 双写缓冲区，innoDB将数据页从缓冲池刷新到磁盘前，先将数据页写入双写缓冲区文件中，便于系统异常时恢复数据
- Redo Log
    - 重做日志，用来实现事务持久性，该日志文件由两部分组成：重做日志缓冲（redo log buffer）以及重做日志文件（redo log），前者在内存中，后者在磁盘中。当事务提交之后会把所有修改信息都存到该日志中，用于刷新脏页到磁盘中、发生错误时，进行数据恢复使用

#### 后台线程

1. Master Thread

    核心后台线程，负责调度其他线程，还负责将缓冲池的数据异步刷新到磁盘中，保持数据一致性，包括脏页刷新、合并插入缓存、undo页的回收

2. IO Thread

    在InnoDB中大量使用了AIO来处理IO请求，这样可以极大地提高数据库性能，而IO Thread主要负责这些IO请求的回调

    - Read thread 4
    - Write thread 4
    - Log thread 1
    - Insert buffer thread 1

3. Purge Thread

    主要用于回收事务已经提交了的undo log

4. Page Cleaner Thread

    协助Master Thread刷新脏页到磁盘的线程，可以减轻Master Thread的工作压力，减少阻塞

#### 事务原理

- 事务是一组操作的集合，是不可分割的工作单位，这些操作要么同时成功要么同时失败。
- 特性ACID

特性的实现

**redo log和undo log**

- 原子性
- 一致性
- 持久性

**锁和MVCC**

- 隔离性

##### redo log

重做日志，记录事务提交时数据页的物理修改，用于实现事务的持久性。

文件由两部分组成，重做日志缓冲和重做日志文件。当事务提交后，会把所有修改信息都存到该日志文件中，用于在刷新脏页到磁盘**发生错误**时进行数据恢复

- WAL write-ahead logging

##### undo log

回滚日志，记录数据被修改前的信息，作用包括提供回滚和MVCC

undo log不同于redo log，属于逻辑日志，其中记录的是实际操作的反向操作，如delete操作会使文件中增加insert记录。

- undo log销毁：事务执行时产生，事务提交时不会立即删除undo log，因为这些日志可能还用于MVCC
- undo log存储：undo log采用段的方式进行管理和记录，存放在rollback segment回滚段中，内部包含1024个undo log segment

##### MVCC

Multi-Version Concurrency Control

- 当前读

    读取的是记录的最新版本，读取时还要保证其他并发事务不能修改当前记录，会对当前读取的记录进行加锁。select...lock in share mode, selcet... for update, update, insert, delete都是当前读

- 快照读

    简单的select（不加锁）就是快照读，读到的是数据的历史版本，有可能是历史数据，不加锁是非阻塞读

    - Read Committed 每次select都生成一个快照读 
    - Repeatable Read 开启事务后第一个select语句才是快照读的地方
    - Serializable 快照读退化为当前读

- MVCC——多版本并发控制，指维护一个数据的多个版本，使得读写操作没有冲突，快照读为MySQL实现MVCC提供了一个非阻塞读功能。MVCC具体实现依赖数据库记录中的**三个隐式字段**、**undo log**、**realView**

###### 隐藏字段

1. DB_TRX_ID

    最近修改事务ID，记录插入这条记录或最后一次修改改记录的事务ID

2. DB_ROLL_PTR

    回滚指针，指向这条记录的上一个版本，用于配合undo log指向上一版本

3. DB_ROW_ID

    隐式主键，表结构本身没有主键会生成

###### undo log

回滚日志，当insert的时候，产生的undo log只在回滚时需要，事务提交后可被立即删除；而update、delete时，产生的undo log不仅在回滚时需要，在快照读时也需要，不会被立即删除。

- undo log版本链

     ![Screenshot 2023-07-13 at 11.08.59](/images/undolog_chain.png)

###### readview

readview（读视图）是快照读执行时MVCC提取数据的依据，记录并维护系统当前活跃的事务（未提交）的id

readview包含四个核心字段：

- m_ids 当前活跃事务id集合
- min_trx_id 最小活跃事务ID
- max_trx_id 预分配事务ID，即当前最大事务ID+1
- creator_trx_id readview创建者的事务ID

版本链数据访问规则：trx_id代表当前版本事务ID

1. trx_id == creator_trx_id 可以访问 （说明数据是当前事务更改的
2. trx_id < min_trx_id 可以访问 （说明数据已经提交了
3. trx_id > max_trx_id 不可以访问 （说明事务是在readview生成后才开启
4. min_trx_id <= trx_id <= max_trx_id 若trx_id不在m_ids中是可以访问的 （说明数据已经提交

不同的隔离级别，生成readview的时机不同：

- Read Committed 在事务中每次执行快照读时生成readview
- Repeatable Read 仅在事务中第一次执行快照读时生成readview，后续复用该readview

结合**undo log**例子和**readview**规则理解
