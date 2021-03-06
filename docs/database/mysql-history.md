## MySQL逻辑架构、并发控制与事务

MySQL最重要最与众不同的特性是它的存储引擎架构，这种架构将查询处理及其它系统任务和数据的存储提取分离。这种处理和存储分离的设计可以在使用时根据性能、特性以及其他需求来选择数据存储的方式。

### 逻辑架构

MySQL架构基本分为三层：

- 最上层的服务层：基于网络的客户端/服务器都有类似的架构，比如连接处理、授权认证、安全等。每个客户端连接都会在服务器进程中拥有一个线程。服务器会负责缓存线程，因此不需要为每个新建的连接创建或者销毁线程。认证基于用户名、原始主机和密码以及用户权限。
- 第二层的核心服务层：包括查询解析、分析、优化、缓存及所有的内置函数，所有跨存储引擎的功能(存储过程、触发器、视图)。MySQL会解析查询，并创建内部数据结构（解析树），然后对其进行各种优化，包括重写查询、决定表的读取顺序，以及选择合适的索引等。用户可以通过特殊的关键字提示（hint）优化器，影响它的决策过程。也可以请求优化器解释（explain）优化过程的各个因素。对于SELECT语句，在解析查询之前，会先检查缓存。
- 存储引擎：负责数据的存储与提取。服务器通过API与存储引擎进行通讯。存储引擎API包含几十个底层函数。

### 并发控制

有两个层次的并发控制：服务器层与存储引擎层。

#### 读写锁
并发控制一般使用锁实现。两种类型的锁，共享锁(shared lock)与排它锁(exclusive lock)，也叫读锁(read lock)和写锁(write lock)。

读锁是共享的，或者说是相互不阻塞的。写锁是排他的。

#### 锁粒度

锁定的数据量越少，则系统的并发程度越高，但消耗的资源也越大。锁策略就是在锁的开销和数据的安全性之间寻求平衡。锁策略有两种：表锁与行级锁

#### 表锁table lock

最基本的锁策略，开销最小，会锁定整张表。读锁互不阻塞，写锁比读锁有更高的优先级。服务器会为ALTER TABLE之类的语句使用表锁

#### 行级锁row lock

行级锁可以最大程度的支持并发，同时也带来了最大的锁开销。行级锁只在存储引擎层实现。

### 事务

事务就是一组原子性的SQL查询，或者说一个独立的工作单元。事务内的语句，要么全部执行，要么全部执行失败。

事务有ACID的标准特征。

#### 原子性atomicity

一个事务必须被视为一个不可分割的最小工作单位，整个事务的操作，要么全部提交成功，要么全部回滚失败。

#### 一致性consistency

数据库总是从一个一致性的状态转换到另外一个一致性的状态。

#### 隔离性isolation

一个事务所做的修改在最终提交之前对其它事务是不可见的

#### 持久性durability

一旦事务提交，则其所做的修改就会永久保存到数据库。

#### 隔离级别

有四种隔离级别：

- READ UNCOMMITTED未提交读：事务中的修改，即使没有提交，对其它事务也是可见的。事务可以读取未提交的数据，称为脏读Dirty Read，实际中很少使用
- READ COMMITTED提交读：大多数数据库的默认隔离级别。一个事务只能看到之前已经提交的事务所做的修改。解决了脏读的问题，但会有不可重复读的问题，执行两次同样的查询，可能会得到不一样的结果。也叫不可重复读nonrepeatable read
- REPEATABLE READ可重复读：MySQL的默认隔离级别。解决了脏读的问题，保证了同一个事务中多次读取同样记录的结果是一致的。但无法解决幻读Phantom Read的问题。幻读指某个事务在读取某个范围内的记录时，另外一个事务又在该范围内插入了新的纪录，当之前的事务再次读取该范围的记录时，会产生幻行Phantom Row。InnoDB和XtraDB存储器通过多版本并发控制MVCC解决了幻读的问题。
- SERIALIZABLE可串行化：最高的隔离级别。通过强制事务串行执行，避免了幻读的问题。会在读取的每一行数据都加上锁，实际很少应用。

#### 死锁
死锁是指两个或多个事务在同一资源上相互占用，并请求锁定对方占用的资源，从而导致恶性循环。

解决方法：

- 复杂的系统比如InnoDB，会检测到死锁的循环依赖，并返回一个错误。
- 查询的时间达到锁等待超时的设定后放弃锁请求，一般不好
- 将持有最好行级排他锁的事务进行回滚，性对比较简单。

死锁发生后，只有部分或完全的回滚其中一个事务，才能打破死锁。大多数情况下只需要重新执行因死锁回滚的事务即可。

#### 事务日志

使用事务日志，存储引擎在修改表时只需要修改其内存拷贝，再把该修改行为记录到持久在硬盘上的事务日志中。事务日志采用追加的方式，是磁盘上一小块区域内的顺序I/O，相对较快。事务日志持久之后，内存中的修改可以慢慢的刷回到磁盘。通常称为预写式日志Write-Ahead Logging，修改数据需要写两次磁盘。

#### 自动提交

MySQL默认采用自动提交模式，如果不是显示地开始一个事务，则每个查询都被当做一个事务提交操作。
```
SHOW VARIABLES LIKE 'AUTOCOMMIT';
SET AUTOCOMMIT = 1;
```
设置当前会话隔离界别：
```
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

#### 隐式和显式锁定

InnoDB采两阶段锁定协议，在事务执行过程中，随时可以执行锁定，锁只有在执行COMMIT或者ROLLBACK时才会释放，而且所有的锁在同一时刻被释放。这都是隐式锁定，InnoDB会根据隔离级别在需要的时候自动加锁。

显式锁定：
```
SELECT ... LOCK IN SHARE MODE
SELECT ... FOR UPDATE
```
MySQL也支持LOCK TABLES和UNLOCK TABLES，这是在服务器层实现的，与存储引擎无关，不能代表事务处理。

LOCK TABLES和事务相互影响，除了事务中禁用了AUTOCOMMIT，可以使用LOCK TABLES，其他时候都不要显式的使用LOCK TABLES。

### 多版本并发控制MVCC

MySQL的大多数事务型引擎实现的都不是简单的行级锁，急于提升并发性能的控制，一般都同时实现了MVCC。MVCC可以视为行级锁的一个变种，在很多情况下避免了加锁操作，开销更低。实现了非阻塞的读操作，写操作也只锁定必要的行。

MVCC通过保存某一时间点的快照实现。InnoDB的MVCC在每行记录后面保存两个隐藏的列，一个保存了行的创建时间，一个保存了行的过期时间(删除时间)。当然保存的并不是实际的时间值，而是系统版本号。系统每开始一个新的事务，系统版本号都会自动递增。

- SELECT：只查找版本早于当前事务版本的数据行，行的删除版本要么未定义，要么大于当前事务版本号
- INSERT：新插入的每一行保存当前事务版本号
- DELETE：每一行保存当前事务版本号当做行删除标识
- UPDATE：插入一行新纪录，保存当前版本号，同时保存当前版本号当做旧行的行删除标识

MVCC只在提交读和可重复读两个隔离级别下工作。

### 存储引擎

查看引擎，表状态：
```
SHOW TABLE STATUS LIKE 'user' \G 
```
#### InnoDB存储引擎

InnoDB存储引擎是MySQL的默认事务型引擎，也是最重要，最广泛的存储引擎。

- 支持事务
- 采用MVCC来支持高并发，实现了四个标准的隔离界别，默认级别是可重复读，并通过间隙锁策略防止幻读的出现。间隙锁不仅仅锁定查询涉及的行，还会对索引中的间隙进行锁定，以防止幻影行的插入。
- 表基于聚簇索引实现。对主键查询有很高的的性能，二级索引(非主键索引)中必须包含主键列
- 很多优化，加速读操作的自适应哈希索引，加速插入操作的插入缓冲区。
- 支持真正的热备份

#### MyISAM存储引擎

5.1之前是默认的存储引擎，不支持事务和行级锁，崩溃后无法安全恢复，对于只读的数据，或者表比较小，可以忍受修复(repair)，可以使用。

- 加锁与并发：对整张表加锁，不支持行级锁。
- 修复：可以手工或者自动执行检查和修复。与事务及崩溃恢复不同。修复可能导致一些数据丢失，而且非常慢
- 索引特性：支持全文索引，基于分词创建的索引
- 延迟更新索引建：创建表时，如果指定了DELAY_KEY_WRITE选项，每次修改执行完成时，不会立刻将修改的索引数据写入磁盘，而是会写到内存中的键缓冲区，只有在清理键缓冲区或者关闭表的时候才会将索引块写入到磁盘。极大提升了写入性能

#### 转换引擎

ALTER TABLE最简单，需要执行很长时间：
```
ALTER TABLE mytable ENGINE=InnoDB;
```
先创建一个新的存储引擎表，然后插入数据：
```
CREATE TABLE innodb_table LIKE myisam_table; 

ALTER TABLE innodb_table ENGINE=InnoDB; 

INSERT INTO innodb_table SELECT * FROM myisam_table;
```
数据量较多时可以分批处理。
