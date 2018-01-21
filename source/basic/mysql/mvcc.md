mysql 事务锁 & mvcc

# 事务-ACID

## 保障数据的故障恢复

* 原子性：整个事务中的所有操作要么全部成功要么全部失败回滚。 事务概念的本质体现。
* 一致性：单考虑某一个事务时（即隔离执行事务时，没有其他事务并发执行的情况下）保持数据库的一致性。数据库从一个一致性的状态转移到另一个一致性的状态。其意义在于保证数据的完整性。
* 隔离性：每个事务间执行相互不影响，感觉不到其他事务的存在而并发执行。意义在于是事务并发控制技术的基础。
* 持久性：一旦事务提交其所做的修改就会永久保存到数据库中，此时即使系统崩溃，修改的数据也不会丢失。意义在于保障数据的故障恢复。

## 事务-分类
1. 隐式事务
 每条单独的语句都是一个事务
1. 显式事务
 每个事务都以start transaction开始，以commit或者rollback结束
 
##  事务隔离级别
读未提交（Read Uncommitted-RU）
读已提交（Read Committed-RC）(mvcc)
可重复读（Repeatable Read-RR）(mvcc)
可串行化（Serializable-S）--加锁读

# #InnoDB-锁策略

### 根据锁的粒度分为

1. 表锁（table lock）
  最基本锁策略
  在服务器层使用
  对整个表加锁，并发性低开销低
1. 行锁（row lock）
  支持最大程度并发，锁开销大
  只在存储引擎层实现
  对某行(索引)记录加上锁

#### 锁是防止其他事务访问指定资源，实现并发控制的手段。

#### InnoDB的锁大致分为:
1. 表锁
 服务器会为诸如: ALTER Table 之类的语句使用表锁,忽略存储引擎的锁机制
 行锁
2. 支持并发高,带来最大的锁开销. 在存储引擎层实现,服务器感知不到 

#### InnoDB还实现了一种锁,叫意向锁(Intention Lock).意向锁是将锁定的对象分为多个层次

1. (1). 意向共享锁(IS Lock) 
2. (2). 意向排他锁(IX Lock)
比如:  需要对页上的记录加X锁,那么需要分别对 数据库A,表,页 上加意向锁IX,最后对记录r上加X锁.
一旦对数据库A,表,页上加IX锁失败,则阻塞.


### 根据锁的类型分为

1. 共享锁（S) 
 由读操作加上的锁，加锁后其他用户只能获取该数据行的共享锁，不能获取排它锁，也就是说只能读不能写
1.   排它锁（X）
由写操作加上的锁，加锁后其他用户不能获取数据行的任何锁

读锁：也叫共享锁、S锁，若事务T对数据对象A加上S锁，则事务T可以读A但不能修改A，其他事务只能再对A加S锁，而不能加X锁，直到T释放A上的S 锁。这保证了其他事务可以读A，但在T释放A上的S锁之前不能对A做任何修改。

写锁：又称排他锁、X锁。若事务T对数据对象A加上X锁，事务T可以读A也可以修改A，其他事务不能再对A加任何锁，直到T释放A上的锁。这保证了其他事务在T释放A上的锁之前不能再读取和修改A。
表锁：操作对象是数据表。Mysql大多数锁策略都支持(常见mysql innodb)，是系统开销最低但并发性最低的一个锁策略。事务t对整个表加读锁，则其他事务可读不可写，若加写锁，则其他事务增删改都不行。

行级锁：操作对象是数据表中的一行。是MVCC技术用的比较多的，但在MYISAM用不了，行级锁用mysql的储存引擎实现而不是mysql服务器。但行级锁对系统开销较大，处理高并发较好

但锁的类型又分为:
(1). 共享锁(S Lock) , 允许事务读取一行数据
(2). 排他锁(X Lock),允许事务删除或更新一行数据.


### 意向锁
意向锁（Intention locks）--封锁意向--表级别锁
The main purpose of IX and IS locks is to show that someone is locking a row, or going to lock a row in the table.

分类

分为意向共享锁（IS）和 意向排它锁（IX）

作用

提高mysql表锁和innodb行锁共存下定位锁冲突的效率
存在价值在于当定位到特定的行所持有的锁之前，提供一种更粗粒度的锁，可以大大节约引擎对于锁的定位和处理的性能（降低封锁成本提高并发性能）

说明

意向锁的存在价值在于在定位到特定的行所持有的锁之前，提供一种更粗粒度的锁，可以大大节约引擎对于锁的定位和处理的性能，因为在存储引擎内部，锁是由一块独立的数据结构维护的，锁的数量直接决定了内存的消耗和并发性能。例如，事务A对表t的某些行修改（DML通常会产生X锁），需要对t加上意向排它锁，在A事务完成之前，B事务来一个全表操作（alter table等），此时直接在表级别的意向排它锁就能告诉B需要等待（因为t上有意向锁），而不需要再去行级别判断。 

申请意向锁的动作是数据库自动完成的，就是说，事务A申请一行的行锁的时候，数据库会自动先开始申请表的意向锁，不需要我们程序员使用代码来申请

意向锁为了方便检测表级锁和行级锁之间的冲突，故在给一行记录加锁前，首先给该表加意向锁。也就是同时加意向锁和行级锁

代码申请共享锁，数据库收到请求后，先尝试申请意向共享锁，成功后，才能成功申请共享锁


### 间隙锁

InnoDB行锁有三种算法

1. record lock
 锁定索引记录本身
1. gap lock
 在索引记录的间隙加锁
 锁定一个范围 但不包含记录本身
1. next key lock
 Record lock和gap lock的结合
 锁定一个范围并锁定记录本身
 防止Phantom Problem
 
说明

InnoDB引擎会自动给会话事务中的共享锁、更新琐以及独占锁，需要加到一个区间值域的时候，再加上个间隙锁（或称范围锁），对不存在的数据也锁住，防止出现幻写。【只在重复读级别下存在，若RC级别则不存在间隙锁】

A gap lock is a lock on a gap between index records, or a lock on the gap before the first or after the last index record. For example, SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 FOR UPDATE; prevents other transactions from inserting a value of 15 into column t.c1, whether or not there was already any such value in the column, because the gaps between all existing values in the range are locked.

INFORMATION_SCHEMA provides access to database metadata, information about the MySQL server such as the name of a database or table, the data type of a column, or access privileges. Other terms that are sometimes used for this information are data dictionary and system catalog.
INFORMATION_SCHEMA Usage Notes
INFORMATION_SCHEMA is a database within each MySQL instance, the place that stores information about all the other databases that the MySQL server maintains. TheINFORMATION_SCHEMA database contains several read-only tables. They are actually views, not base tables, so there are no files associated with them, and you cannot set triggers on them. Also, there is no database directory with that name.
Although you can select INFORMATION_SCHEMA as the default database with a USE statement, you can only read the contents of tables, not perform INSERT, UPDATE, orDELETE operations on them.


# MVCC-Multiversion Concurrency Control
*  核心
 数据库行锁与行的多个版本结合起来，只需很小开销，实现非锁定读
*  原理
 通过保存数据在某个时间点的快照来实现
 
### 三个重要文件 
* Redo log
保存指定的sql语句到指定的文件(ib_logfile0, ib_logfile1)
recovery时重新执行即可

* Undo log
为回滚及可持续读之用(idb表空间)
copy事务前的数据行内容到undo buffer，然后刷新到磁盘

* Rollback Segment
undo log被划分为多段
具体某行的undo log保存在某个段中

说明

redo log在磁盘上作为一个独立的文件存在，即Innodb的log文件。
redo log不同的是，磁盘上不存在单独的undo log文件，所有的undo log均存放在主ibd数据文件中（表空间

mvcc只在RC和RR隔离级别下工作，其他两个级别和mvcc都不兼容，RC总是读取最新数据的行，不符合当前事务版本的数据行，而S会对所有读取的行都加锁。

MVCC (Multiversion Concurrency Control)，即多版本并发控制技术,它使得大部分支持行锁的事务引擎，不再单纯的使用行锁来进行数据库的并发控制，取而代之的是把数据库的行锁与行的多个版本结合起来，只需要很小的开销,就可以实现非锁定读，从而大大提高数据库系统的并发性能



### 实现
InnoDB存储每行记录的后面都内置了三个字段

* 6个字节的DB_TRX_ID
事务id编号标记最新更新该行记录的transaction id，每处理一个事务，其值自动+1
* 7个字节的DB_ROLL_PTR
回滚段指针指向当前记录项的rollback segment的undo log记录，找之前版本的数据就是通过这个指针
* 1个DELETE bit位
用于标识该记录是否被删除

如下：
Primary key1
Primary key2
Other columns
DB_TRX_ID
6*8bit
DB_ROLL_PTR
7*8bit
DELETED
1bit

举个栗子:

有这样的数据行
Row Record
10
20
30
40
50
DB_TRX_ID
DB_ROLL_PTR
DELETED

（1）事务1更新该行字段的值

Transaction1

11
21
31
41
51
02
0x1231
0

Undo log

10
20
30
40
50
01
null
0

当事务1更改该行的值时，会进行如下操作：
用排他锁锁定该行
记录redo log
把该行修改前的值Copy到undo log，通过回滚指针与主数据关联
修改当前行的值，填写事务编号，使回滚指针指向undo log中的修改前的行

F1～F6是某行列的名字，1～6是其对应的数据。后面三个隐含字段分别对应该行的事务号和回滚指针，假如这条数据是刚INSERT的，可以认为ID为1，其他两个字段为空。


### 具体操作过程
* Select
select同时满足以下两个条件的行才会返回
版本早于当前事务版本的数据行(也就是数据行的版本必须小于等于当前事务的版本)，这确保当前事务读取的行都是事务之前已经存在的，或者是由当前事务创建或修改的行
行的删除操作的版本一定是未定义的或者大于当前事务的版本号，确定了当前事务开始之前，行没有被删除
* Insert
InnoDB为新插入的每一行保存当前系统版本号作为行的事务版本号
* Delete
InnoDB为删除的每一行保存当前系统版本号作为行的版本号，同时置DELETED为1
* Update
InnoDB复制并插入一行，使用当前系统版本号作为新行的版本号，同时保存当前系统版本号到原来的行作为行删除标识

通过MVCC，虽然每行记录都需要额外的存储空间，更多的行检查工作以及一些额外的维护工作，但可以减少锁的使用，大多数读操作都不用加锁，读数据操作很简单，性能很好，并且也能保证只会读取到符合标准的行，也只锁住必要行

### mvcc在RC与RR下的区别
* RC下，每次select都会去读最新的版本（快照）
  读取被锁定行的最新一份快照数据.  产生了不可重复读的问题

* RR下，select一直使用当前事务最开始读到的版本（快照）
 读取事务开始时的行数据版本.  解决不可重复读的问题

说明

读取历史数据的方式，我们叫它快照读 (snapshot read)，而读取数据库当前版本数据的方式，叫当前读 (current read)。很显然，在MVCC中：
快照读：就是select
select * from table ....;
当前读：特殊的读操作，插入/更新/删除操作，属于当前读，处理的都是当前的数据，需要加锁。

* select * from table where ? lock in share mode;
* select * from table where ? for update;
* insert;
* update ;
* delete;

事务的隔离级别实际上都是定义了当前读的级别，MySQL为了减少锁处理（包括等待其它锁）的时间，提升并发能力，引入了快照读的概念，使得select不用加锁。而update、insert这些“当前读”，就需要另外的模块来解决了。

### 特点
1. undo log中的行就是MVCC中的多版本
1. 读写不冲突, 实现非阻塞读
 * 任何操作都不会阻塞读
 * 读操作也不会阻塞任何操作，因为读不加锁
1. 实现可重复读
1. mvcc只在RC与RR级别下工作

mvcc只在RC和RR隔离级别下工作，其他两个级别和mvcc都不兼容，RU总是读取最新数据的行，不符合当前事务版本的数据行，而S会对所有读取的行都加锁。

MVCC (Multiversion Concurrency Control)，即多版本并发控制技术,它使得大部分支持行锁的事务引擎，不再单纯的使用行锁来进行数据库的并发控制，取而代之的是把数据库的行锁与行的多个版本结合起来，只需要很小的开销,就可以实现非锁定读，从而大大提高数据库系统的并发性能

### 一致性非锁定读

InnoDB存储引擎通过行多版本控制方式读取当前执行的时间数据库中行的数据

不加锁读

* InnoDB下读取数据的方式（RC/RR）
* 快照数据
* 基于MVCC机制，select操作是在读取undo中历史数据，因为历史数据不需要修改所以不需要加任何锁
*  InnoDB实现非阻塞读的实现.极大的提高数据库的并发性

### 一致性锁定读
* 不使用undo日志，直接对当前记录加锁
* 1). select .... for update. 加X锁
* 2). select .... lock in share mode. 加S锁













