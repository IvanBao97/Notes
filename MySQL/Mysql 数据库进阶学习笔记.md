# Mysql 数据库进阶学习笔记

> 本篇笔记参考了：
>
> 1. 《MySQL 官方文档》
> 2. 《高性能MySQL 第3版》
> 3. 《MySQL技术内幕 : InnoDB存储引擎 第2版》
> 4. 《MySQL 进阶》— 黑马程序员 
> 5. 《mysql 查询流程解析及重要知识总结》— 斜阳雨陌博客
> 6. 《EXPLAIN的参数解析及简单应用》— 环游记博客
> 7. 《彻底搞清分库分表》— 传智燕青



TODO：

1. 日志文件
2. 多版本并发控制
3. 主从复制



## 1. MySQL 逻辑架构

### 1.1 架构分层

[![image-MySQL架构](https://camo.githubusercontent.com/93724a49f5bba4e13c129caecaa62929be096861cf63edc807c8f1bc5db86699/68747470733a2f2f696d672d626c6f672e6373646e2e6e65742f32303138303833313137333931313939373f77617465726d61726b2f322f746578742f6148523063484d364c7939696247396e4c6d4e7a5a473475626d56304c337066636e6c6862673d3d2f666f6e742f3561364c354c32542f666f6e7473697a652f3430302f66696c6c2f49304a42516b46434d413d3d2f646973736f6c76652f3730)](https://camo.githubusercontent.com/93724a49f5bba4e13c129caecaa62929be096861cf63edc807c8f1bc5db86699/68747470733a2f2f696d672d626c6f672e6373646e2e6e65742f32303138303833313137333931313939373f77617465726d61726b2f322f746578742f6148523063484d364c7939696247396e4c6d4e7a5a473475626d56304c337066636e6c6862673d3d2f666f6e742f3561364c354c32542f666f6e7473697a652f3430302f66696c6c2f49304a42516b46434d413d3d2f646973736f6c76652f3730)

存储引擎架构分为三层，自上而下，分为 **第一层：连接层；第二层：服务层；第三层：引擎层**

这种架构将：**查询处理、其他系统任务、数据的存储与提取** 三部分分离。所以，带来的好处就是可以在使用时**根据性能、特性，以及其他需求来选择数据存储方式**。

#### 连接层

MySQL的最上层是连接服务，引入了线程池的概念，允许多台客户端连接。主要工作是：**连接处理、授权认证、安全防护等**。

连接层为通过安全认证的接入用户提供线程，同样，在该层上可以实现基于SSL 的安全连接。

#### 服务层

服务层用于处理核心服务，如标准的SQL接口、查询解析、SQL优化和统计、全局的和引擎依赖的缓存与缓冲器等等。所有的与存储引擎无关的工作，如过程、函数等，都会在这一层来处理。

在该层上，服务器会解析查询并创建相应的内部解析树，并对其完成优化，如确定查询表的顺序，是否利用索引等，最后生成相关的执行操作。如果是SELECT 语句，服务器还会查询内部的缓存。如果缓存空间足够大，这样在解决大量读操作的环境中能够很好的提升系统的性能。

#### 引擎层

存储引擎层，存储引擎负责实际的MySQL数据的**存储与提取**，**服务器通过API 与 存储引擎进行通信**。



### 1.2 MySQL 工作流程

[![image-MySQL工作流程](https://camo.githubusercontent.com/d8376c3fc78d4e29eaab5968884c8dcf6133b3076a5bd8714bdf044faeefbde9/68747470733a2f2f696d672d626c6f672e6373646e696d672e636e2f32303139313231353137353135303235382e706e673f782d6f73732d70726f636573733d696d6167652f77617465726d61726b2c747970655f5a6d46755a33706f5a57356e6147567064476b2c736861646f775f31302c746578745f6148523063484d364c79397462334a30655335696247396e4c6d4e7a5a473475626d56302c73697a655f31362c636f6c6f725f4646464646462c745f3730)](https://camo.githubusercontent.com/d8376c3fc78d4e29eaab5968884c8dcf6133b3076a5bd8714bdf044faeefbde9/68747470733a2f2f696d672d626c6f672e6373646e696d672e636e2f32303139313231353137353135303235382e706e673f782d6f73732d70726f636573733d696d6167652f77617465726d61726b2c747970655f5a6d46755a33706f5a57356e6147567064476b2c736861646f775f31302c746578745f6148523063484d364c79397462334a30655335696247396e4c6d4e7a5a473475626d56302c73697a655f31362c636f6c6f725f4646464646462c745f3730)

**第一层：建立连接**

1. 连接处理：客户端同数据库服务层建立 TCP 连接，连接层会请求一个连接线程。如果连接池中有空闲的连接线程，则分配给这个连接，如果没有，在没有超过最大连接数的情况下，创建新的连接线程负责这个客户端。
2. 授权认证：在真正的操作之前，还需要调用用户模块进行授权检查，来验证用户是否有权限。通过后，连接线程开始接收并处理来自客户端的 SQL 语句。

**第二层：核心服务**

1. 连接线程接收到 SQL 语句之后，将语句交给 SQL 语句解析模块进行语法分析和语义分析。
2. 如果是一个 SELECT 语句，则先查询缓存，如果有结果直接返回给客户端。
3. 如果查询缓存中没有结果，再查询数据库引擎层。先通过 SQL 优化器（Optimizer），进行查询的优化。

**第三层：数据库引擎层**

1. 打开表，如果需要的话获取相应的锁。
2. 先查询 **缓存页** 中有没有相应的数据，如果有则可以直接返回，如果没有就要从磁盘上去读取。
3. 当在磁盘中找到相应的数据之后，则会加载到缓存中来，从而使得后面的查询更加高效，由于内存有限，多采用变通的LRU表来管理缓存页。

最后，获取数据后返回给客户端，关闭连接，释放连接线程。





## 2. MySQL 存储引擎

### 2.1 MyISAM 和 InnoDB 的区别

| 对比项 | MyISAM                     | InnoDB                           |
| ------ | -------------------------- | -------------------------------- |
| 主外键 | 不支持                     | 支持                             |
| 事务   | 不支持                     | 支持                             |
| 锁     | 表锁                       | 行锁                             |
| 缓存   | 只缓存索引，不缓存真实数据 | 缓存索引和真实数据，对内存要求高 |
| 表空间 | 小                         | 大                               |
| 关注点 | 性能                       | 事务                             |

- MyISAM

  - B+ Tree 索引文件和数据文件是分离的**（非聚集索引）**

  ![image-MyISAM](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/MyISAM.png)

- InnoDB

  - 表数据文件本身就是按 B+ Tree 组织的一个索引结构文件**（聚集索引）**
  - 叶节点包含了完整的数据记录

  ![image-InnoDB](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/InnoDB.png)

  - InnoDB 表必须建立主键，并且推荐用整型的自增主键
    1. 建立主键是为了方便使用主键索引去维护 B+ Tree
       - 如果没有建立主键，MySQL 会自动寻找一个数据不重复的字段列作为唯一索引，来维护 B+ Tree
       - 如果没有找到数据不重复的字段，MySQL 会自动添加一个唯一字段维护 B+ Tree
    2. 使用整型是因为方便查找时索引比较，且占用空间小
    3. 自增可以保证，新增元素可以直接插在叶子节点的最右边，从而不改变其他部分结构，因为叶子节点的索引值，从左到右依次增加

### 2.2 InnoDB 引擎结构

[![image-InnoDBStructure](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/InnoDB%E5%BC%95%E6%93%8E%E7%BB%93%E6%9E%84.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/InnoDB引擎结构.png)

#### 2.2.1 线程

- Master Thread
  - 核心后台线程，负责将缓冲池中的数据异步刷新到磁盘，保证数据的一致性（包扩了下面三个线程的全部功能）
- IO Thread
  - Write Theard 写线程
  - Read Theard 读线程
  - Insert buffer Theard 处理插入缓冲线程
  - Log Theard 日志操作线程
- Purge Thread
  - 用来回收事务提交后，不再使用的 undo Log
- Page Cleaner Theard
  - 将内存缓冲池里的脏页刷新到磁盘文件

#### 2.2.2 内存

- 缓冲池
  - 作用：
    - 通过内存的速度来弥补磁盘速度较慢对数据库性能的影响
  - 读流程：
    - 进行读取页操作时，首先判断该页是否在缓冲池中，若存在，则命中；不存在，则从磁盘读取，并将该页存放在缓冲池中
  - 写流程：
    - 首先修改缓冲池中的页，然后以一定的频率刷新到磁盘上（并不是每次修改都出发刷新，而是通过 CheackPoint 机制刷新回磁盘）
  - CheackPoint 机制
    - 事务操作在写之前，会先记录 redo log，当之前的 redo log 过期时，将其记录的操作页刷回磁盘
    - LRU 淘汰页是脏页时，会触发 CheackPoint，将脏页刷回磁盘
  - 管理方式（Data Page）：
    - LRU with midPoint
      - 新访问的页放在 midPoint 而不是首部，防止由于全扫描使一些非热点数据页被插在首部，导致热点数据移除
    - Free List：
      - 存储空闲页。每当 LRU 需要加入新页时，先查询 Free List ，有空闲页，将新页直接插入 LRU，并删除一个空闲页；没空闲，采用 LRU 淘汰策略删除非热点页
    - Flush List：
      - LRU 中的页被修改后，被称为脏页，等待 CheckPoint 机制刷回磁盘。Flush List 存储了所有的脏页，管理脏页的刷新。同时，用户依旧可以在 LRU 中查看修改的页

#### 2.2.3 关键特性

- 插入缓冲（Insert Buffer）：

  - 对于非聚簇索引列插入数据时，因为不是自增列，会导致磁盘随机访问，降低性能
  - InnoDB 设置了插入缓冲，先在内存中预合并这些插入数据，再定期刷回磁盘，优化了非聚簇索引列数据的插入性能

- 两次写（Double Write）：

  - 解决页损坏时，数据丢失的问题（记录损坏可以通过 redo log 恢复）
  - 过程：先写到 Double Write Buffer，再分两次写到磁盘共享空间，最后再从 Buffer 写到磁盘文件

- 自适应哈希索引（Adaptive Hash Index）：

  - InnoDB 存储引擎会监控各表索引页的查询，自动根据访问的频率和模式为某些热点页建立哈希索引

  - 条件：连续对某一模式访问100次 或 访问次数达到 N 次（N = 页记录 / 16）

  - Eg：对于(a, b) 这种联合索引，访问模式有以下两种：

    ```mysql
    WHERE a = xxx;
    WHERE a = xxx AND b = xxx;
    ```

- 异步IO：

  - 通过 AIO 的方式同时扫描多个页，也可以将多个 IO 操作合并成一个

- 刷新临近页：

  - 当刷新一个脏页时，InnoDB 会检测该页所在区（extent）的所有页，如果是脏页，则一起刷新

#### 2.2.4 文件

- 配置文件、pid 文件（记录当前 MySQL 实例进程ID文件）、表结构定义文件（定义了表结构，后缀为frm）...
- 日志文件
  - 错误日志（主机名.log）：
    - 记录了 MySQL 启动、运行、关闭时遇到的错误信息
  - 慢查询日志（slow_log 表）：
    - 记录了所有慢查询 SQL 语句信息（可以通过设置 long_query_time 来改变慢查询判定的阈值）
  - 查询日志（general_log 表）：
    - 记录了所有查询 SQL 语句信息，以及 Access denied 的请求
  - 二进制日志（bin_log）：【==后续会和事务日志对比详细介绍==】
    - 记录了更改操作 SQL 语句信息，用来恢复数据、复制等
    - 每次事务的操作都会被记录到二进制缓存中，当事务提交时，刷到二进制文件中

#### 2.2.5 数据存储结构

[![image-tableStructure](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/tableStructure.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/tableStructure.png)

[![image-pageStructure](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/PageStructure.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/PageStructure.png)





## 3. Mysql 并发控制

### 3.1 锁

#### 3.1.1 锁的分类及特点

> 不同存储引擎的锁实现也不同，比如 MyISAM 只提供表锁，而 InnoDB 则还支持一致性非锁定读和行锁。
>
> 本笔记是基于 InnoDB 引擎展开的深入学习

**Latch 和 Lock 的区别**

- Latch 是轻量级锁，可分为 mutex（互斥量）和 rwlock（读写锁）。目的是保证并发县城操作资源的正确性，锁定时间短，通常没有死锁检测机制。
- Lock 的对象是事务，用来锁数据库的表、页、行。一般在事务的 commit 和 rollback 之后释放，有死锁检测机制。

|          | Lock                           | Latch                        |
| -------- | ------------------------------ | ---------------------------- |
| 对象     | 事务                           | 线程                         |
| 保护     | 表、页、行                     | 内存数据结构                 |
| 持续时间 | 整个事务过程                   | 临界资源                     |
| 模式     | 行锁、表锁、意向锁             | 读写锁、互斥量               |
| 死锁     | 有（通过 time out 等机制处理） | 无（可通过加锁顺序避免死锁） |
| 存在于   | Lock Manager 的哈希表          | 数据结构的对象中             |

**行锁**

- 共享锁（S锁、读锁）：

  -  允许一个事务去读一行，阻止其他事务获得相同数据集的排他锁。即多个客户可以同时读取同一个资源，但不允许其他客户修改。

  ```MYSQL
  SELECT * FROM xxx WHERE column_1 = xxx LOCK IN SHARE MODE
  ```

- 排他锁（X锁、写锁）：

  - 允许获得排他锁的事务更新数据，阻止其他事务取得相同数据集的读锁和写锁。写锁是排他的，写锁会阻塞其他的写锁和读锁。

  ```MYSQL
  SELECT * FROM xxx FOR UPDATE;
  ```

  

**表锁（数据库隐式帮我们做了，不需要程序员关心）**

- 意向共享锁：事务打算给数据行加行共享锁，事务在给一个数据行加共享锁前必须先取得该表的IS锁。
- 意向排他锁：事务打算给数据行加行排他锁，事务在给一个数据行加排他锁前必须先取得该表的IX锁。



#### 3.1.2 记录锁、间隙锁、临键锁

**【基本概念】**

MySQL 默认的事务隔离级别是可重复读（Repeatable Read），这种隔离级别可能会出现幻读的问题，为了解决这个问题，MySQL 会将数据和其之间的间隙也加锁，这种就叫做临键锁



**记录锁：**封锁某一条记录，记录锁也叫行锁

**间隙锁：**封锁索引记录中的间隔，或者第一条索引记录之前的范围，又或者最后一条索引记录之后的范围

**临键锁：**不仅封锁记录，还封锁记录之间的间隙（记录锁+间隙锁）

**幻读**：是指在一个事务内读取到了别的事务插入的数据，导致前后读取不一致。和不可重复读类似，但幻读会读到其他事务的插入的数据，导致前后读取不 一致，幻读的重点在于新增或者删除（数据条数变化）。



**【基本原则】**

原则 ：加锁的基本单位是 next-key lock。next-key lock 是前开后闭区间。

优化 1：索引上的等值查询，如果命中记录，给唯一索引加锁的时候，next-key lock 退化为行锁。

优化 2：索引上的等值查询，向右遍历时，区间上最后一个值不满足等值条件，next-key lock 退化为间隙锁。

注意：

* 唯一索引只有锁住多条记录或者一条不存在的记录的时候，才会产生间隙锁，指定给某条存在的记录加锁的时候，只会加记录锁，不会产生间隙锁；

* 普通索引不管是锁住单条，还是多条记录，都会产生间隙锁；



**【案例分析】**

```mysql
-- 构建测试表和数据
CREATE TABLE `t` (
  `id` int(10) NOT NULL,
  `num` int(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `index_num` (`num`)
) ENGINE=InnoDB;
INSERT INTO t VALUES (0,0),(5,5),(10,10),(15,15),(20,20),(25,25);

-- 此时MySQL将数据分为了 (-∞，5]，(5，10]，(10，15]，(15，20]，(20，25]，(25，+supernum] 这几个临键锁区间


# 1.唯一索引等值查询
SELECT * FROM t WHERE id = 7 FOR UPDATE; # Transcation A：落在(5，10]临键锁区间，因为id=7，区间降级为间隙锁(5，10)
INSERT INTO t VALUES (4,4); # Transcation B 成功
INSERT INTO t VALUES (6,6); # Transcation C 失败
UPDATE t SET id = 11 WHERE id = 10; # Transcation D 成功

SELECT * FROM t WHERE id = 10 FOR UPDATE; # Transcation A：落在(5，10]，(10，15]临键锁区间，因为id=10且唯一索引命中数据行，区间降级为记录锁，只锁id=5这一行
INSERT INTO t VALUES (4,4); # Transcation B 成功
INSERT INTO t VALUES (6,6); # Transcation C 成功
UPDATE t SET id = 11 WHERE id = 10; # Transcation D 失败


# 2.普通索引等值查询
SELECT * FROM t WHERE num = 5 FOR UPDATE; # Transcation A：落在(-∞，5]，(5，10]临键锁区间，因为非唯一索性，无法降级为记录锁，且根据原则2，降级为(0,5],(5,10)
INSERT INTO t VALUES (4,4); # Transcation B 失败
INSERT INTO t VALUES (6,6); # Transcation C 失败
UPDATE t SET id = 11 WHERE num = 10; # Transcation D 成功
```





#### 3.1.3 死锁

- **死锁产生：**

- - 死锁是指两个或多个事务在同一资源上相互占用，并请求锁定对方占用的资源，就可能产生死锁。

- **检测死锁：**

  - 数据库系统实现了各种死锁检测和死锁超时的机制。InnoDB 存储引擎能检测到死锁的循环依赖并立即返回一个错误。

- **死锁恢复：**

  - 死锁发生以后，只有部分或完全回滚其中一个事务，才能打破死锁。
  - InnoDB 目前处理死锁的方法是，将持有最少行级排他锁的事务进行回滚。

- **外部锁的死锁检测：**

  - 发生死锁后，InnoDB 一般都能自动检测到，并使一个事务释放锁并回退，另一个事务获得锁，继续完成事务。
  - 但在涉及外部锁，或涉及表锁的情况下，InnoDB 并不能完全自动检测到死锁， 这需要通过设置锁等待超时参数 innodb_lock_wait_timeout 来解决

- **避免死锁**：

  - 在事务中，如果要更新记录，应该直接申请足够级别的锁，即排他锁，而不应先申请共享锁、更新时再申请排他锁，因为这时候当用户再申请排他锁时，其他事务可能又已经获得了相同记录的共享锁，从而造成锁冲突，甚至死锁
  - 如果事务需要修改或锁定多个表，则应在每个事务中以相同的顺序使用加锁语句。 在应用中，如果不同的程序会并发存取多个表，应尽量约定以相同的顺序来访问表，这样可以大大降低产生死锁的机会



### 3.2 多版本并发控制



### 3.3 事务

#### 3.3.1 事务的 ACID 特性和隔离级别

##### 3.3.1.1 ACID 特性

- **原子性（Atomicity）**
  - 事务必须被视为一个不可分割的最小工作单元。其中的所有操作要么都提交成功，要么都失败回滚。
  - 通过 undo log 实现。（详见事物的实现篇）
- **一致性（Consistency）**
  - 事务必须将数据库从一种一致状态转变为另一种一致状态，数据库的完整性不会被破坏。
  - 数据库通过原子性、隔离性、持久性来保证一致性。C(一致性)是目的，A(原子性)、I(隔离性)、D(持久性)是手段。
- **隔离性（Isolation）**
  - 每个事务提交前，对其他事务都不可见。
  - 通常用锁和 MVCC 来实现。（详见锁篇）
- **持久性（Durability）**
  - 事务一旦提交，其操作结果就是永久性的。
  - 通过 redo log 实现。（详见事物的实现篇）

##### 3.3.1.2 隔离级别

- 读未提交（Read Uncommitted）
  - 读未提交，任何操作都不加锁，所以能读到其他事务修改但未提交的数据行，也称之为脏读（Dirty Read）。
- 读已提交（Read Committed）
  - 读操作不加锁，写操作加锁。读被加锁的数据时，读事务每次都读 undo log 中的最近版本，因此可能对同一数据读到不同的版本（不可重复读），但能保证每次都读到最新的数据（事务提交之后的，不可重复读，两次读不一致），但是不会在记录之间加间隙锁，所以允许新的记录插入到被锁定记录的附近，所以再多次使用查询语句时，可能得到不同的结果。
- 可重复读（Repeatable Read - InnoDB 引擎默认）
  - 第一次读数据的时候就将数据加行锁（共享锁），使其他事务不能修改当前数据，即可实现可重复读。但是不能锁住 insert 进来的新的数据，当前事务读取或者修改的同时，另一个事务还是可以 insert 提交，造成幻读；
  - 注：mysql 的可重复读的隔离级别解决了 “不可重复读” 和 “幻读” 2 个问题，因为使用了间隙锁。
- 串行化（Serializable）
  - InnoDB 锁表，读锁和写锁阻塞，强制事务串行执行，解决了幻读的问题

**隔离级别与问题对应表如下：**

| 隔离级别                     | 脏读（Dirty Read） | 不可重复读（NonRepeatable Read） | 幻读（Phantom Read） |
| ---------------------------- | ------------------ | -------------------------------- | -------------------- |
| 未提交读（Read uncommitted） | 可能               | 可能                             | 可能                 |
| 已提交读（Read committed）   | 不可能             | 可能                             | 可能                 |
| 可重复读（Repeatable read    | 不可能             | 不可能                           | 可能                 |
| 可串行化（Serializable ）    | 不可能             | 不可能                           | 不可能               |

- SQL 和 SQL2 标准的默认事务隔离级别是 SERIALIZABLE
- InnoDB 存储引擎默认支持的隔离级别是 REPEATABLE READ
  - 但与标准 SQL 不同的是：通过使用 Next-Key Lock 锁的算法来避免幻读的产生
  - 即 InnoDB 在默认的默认隔离级别下已经能完全保证事务的隔离性要求（达到 SQL 标准的 SERIALIZABLE 级别)
- 在 SERIALIZABLE 的事务隔离级别，InnoDB 会对每个 SELECT 语句后自动加上 LOCK IN SHARE MODE,即共享读锁
  - 因此在此隔离级别下，读占用了锁，对一致性的非锁定读不再予以支持
  - 此隔离级别复核数据库理论上的要求，即事务是 well-formed 的，并且是 two-phrased
  - SERIALIZABLE 的隔离级别主要用于 InnoDB 存储引擎的分布式事务
- 在 READ COMMITTED 的事务隔离级别下，除唯一性约束检查及外键约束检查需要 gap lock，其他情况都不会使用

#### 3.3.2 事务的分类

> MySQL 的 InnoDB 引擎是不支持嵌套事务的，在开启了一个事务的情况下，再开启一个事务，会隐式的提交上一个事务。
>
> Mysql 的 InnoDB 引擎，默认是 autocommit = 1，也就是说默认是立即提交，如果想开启事务，先设置 autocommit = 0，然后用 START TRANSACTION、 COMMIT、 ROLLBACK 来使用具体的事务。

##### 3.3.2.1 扁平事务

```mysql
-- 在扁平事务中，所有操作都处于同一层次，其由BEGIN WORK开始，由 COMMIT WORK 或 ROLLBACK WORK 结束，其间的操作要么都执行，要么都回滚。
-- 缺点：不能提交或者回滚事务的某一部分，或分几个步骤提交。
BEGIN WORK
	Operation 1;
	Operation 2;
	...
	Operation k;
COMMIT WORK
```

##### 3.3.2.2 带有保存点的扁平事务

```mysql
-- 隐式的设置了保存点，允许在事务执行过程中回滚同一事务中较早的一个状态
-- 缺点：系统崩溃时，保存点会消失，导致事务回滚后从头开始执行
BEGIN WORK
	Operation A;
	Operation B;
SAVE WORK: 1
	Operation C;
SAVE WORK: 2
	Operation D;
ROLLBACK WORK 1 / ROLLBACK WORK 2
```

##### 3.3.2.3 链事务

- 是带有保存点事务的变种，在提交一个事务时，会释放下一个事务不需要的对象（锁），并将自己的执行结果隐式地传递给下一个事物。
- 链事务与带有保存点事务的区别：
  - 链事务只能回滚到前一个事务的结果处，而带有保存点事务可以回滚到任意一个保存点处
  - 链事务的每个事务在执行完 COMMIT 后会释放自己所持有的锁，而带有保存点事务则会一直持有每个阶段的所，直至最终COMMIT

[![image-链事务](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/ChainTransaction.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/ChainTransaction.png)

##### 3.3.2.4 嵌套事务

```mysql
-- 父事务中嵌套着子事务（并发事务）
BEGIN WORK
	Transaction 1;
	Transaction 2;
	...
	Transaction k;
COMMIT WORK

-- 测试在 InnoDB 引擎中使用嵌套事务
SELECT * FROM test;  
+-----+  
|  n  |  
+-----+  
|  1  |  
+-----+   
  
START TRANSACTION;

INSERT INTO test VALUES(2);  

START TRANSACTION; 
  
INSERT INTO test VALUES(3); 
  
COMMIT;  

ROLLBACK;  

SELECT * FROM test;  
+-----+  
|  n  |  
+-----+  
|  1  |  
|  2  |  
|  3  |  
+-----+  

-- MySQL 的 InnoDB 引擎是不支持嵌套事务的，在开启了一个事务的情况下，再开启一个事务，会隐式的提交上一个事务。
```

##### 3.3.2.5 分布式事务

- 在分布式环境下的扁平事务，在调用多台数据的情况下，同样需要满足事务的 ACID 特性。（详细介绍见 **4.5 MySQL 其他特性**）



#### 3.4 事务的控制语句

```mysql
BEGIN|START TRANSACTION
-- 显式的开启一个事务

COMMIT|COMMIT WORK
-- 都用来提交事务，不同的在于 COMMIT WORK 用来控制事务提交后的行为是 CHAIN 还是 RELEASE 的。如果是 CHAIN 方式，那么事务就变成了链事务。可以通过参数 completion_type 来进行控制，默认为 0 表示没有任何操作，这种时候 COMMIT 和 COMMIT WORK 是完全等价的。为 1 时 COMMIT WORK 等同于 COMMIT AND CHAIN,表示马上开启一个相同隔离级别的事务。为 2 时 COMMIT WORK 等同于 COMMIT AND RELEASE，当事务提交后自动与服务器断开连接。

ROLLBACK|ROLLBACK WORK
-- 都用来回滚事务。结束正在执行的事务，并撤销所有还没提交的修改。


 
SAVEPOINT identifier
-- SAVEPOINE 允许在事务当中创建一个保存点，一个事务中可以创建多个保存点。

RELEASE SAVEPOINT identifier
-- 删除事务的一个保存点，当没有一个该保存点时抛出异常。

ROLLBACK TO [SAVEPOINT] identifier
-- 与 SAVEPOINT identifier 配合使用，用来将事务回滚到某个 identifier 的状态。

SET TRANSACTION
-- 设置事务的隔离级别，InnoDB 中提供了 READ UNCOMMITTED，READ COMMITTED，REPEATABLE READ，SERIALIZABLE 四种隔离级别。InnoDB 默认为 REPEATABLE READ。
```



#### 3.5 使用事务的注意点

- **在循环中提交事务**

  - 当我们需要在循环操作中提交事务时，就需要根据实际情况来选择事务范围
    - 每循环一次，做一次事务的提交。
      - 效率低，但回滚代价小
    - 所有循环操作，在一个事务里提交。
      - 效率高，但回滚代价大

- **自动提交事务或自动回滚**

  - InnoDB 引擎是默认开启事务自动提交的，我们可以选择关闭自动提交

    ```
    SET autocommit = 0 -- 关闭自动提交事务
    ```

  - 事务的提交和回滚时机可以交由程序开发者来处理，从而方便根据业务和需求设计不同的事务提交模式，同时，在发生错误回滚时，也可以得知发生错误的原因和定位。

- **运行长事务**

  - 问题：长事务由于执行过程较长，很可能在执行过程中遇到数据库、操作系统或硬件的问题，使得事务回滚，而重新开始执行就会使得代价过大。
  - 解决：将长事务转化为小批量的事务处理
    - 每完成一个小事务，都将完成的结果存在 batchContext 中，记录完成批量事务中的最大账号 ID，如果遇到问题回滚，我们只需要找到已完成的最大事务 ID，继续执行，从而减少重新执行的代价。

<br>

<br>

## 4. MySQL 性能优化

### 4.1 Schema 与数据类型的优化

#### 4.1.1 数据类型的设计

【**基本原则**】：

1. 更小的通常更好
   - 在满足存储需求的前提下，更小的数据类型会占用更少的磁盘、内存和 CPU 缓存，CPU 处理周期也就越短
2. 越简单的数据类型越好
   - 简单的数据类型通常需要更少的 CPU 处理周期，例如整形比字符操作代价更低，因为字符集和校对规则（排序规则）会使字符操作更复杂
3. 尽量避免 NULL
   - 如果不是需要存储 NULL 值得情况下，最好将列定为 NOT NULL。因为如果查询中包含可为 NULL 的列，会使索引、索引统计和值比较变得更复杂
     - 索引记录会额外使用一个字节来记录 NULL

##### 4.1.1.1 实数类型

**【整数类型】**

| 类型      | 存储空间大小 (n) | 范围 (-2 ^(n-1)^ ~ 2 ^(n-1)^ - 1) |
| --------- | ---------------- | --------------------------------- |
| TINYINT   | 8 位             | -128 ~ 127                        |
| SMALLINT  | 16 位            | -2 ^15^ ~ 2 ^15^ - 1              |
| MEDIUMINT | 24 位            | -2 ^23^ ~ 2 ^23^ - 1              |
| INT       | 32 位            | -2 ^31^ ~ 2 ^31^ - 1              |
| BIGINT    | 64 位            | -2 ^63^ ~ 2 ^63^ - 1              |

UNSIGNED 属性：表示不允许负值，可以使正数上限扩大一倍

- Eg：TINYINT UNSIGNED 0 ~ 255， TINYINT -128 ~ 127

为整数类型设置宽度是没有意义的，它并不会限制实际的合法范围。宽度只是控制类数据的展示个数，对于存储和计算没有影响

**【小数类型】**

| 类型    | 存储空间大小 | 计算类型 | 计算代价                   |
| ------- | ------------ | -------- | -------------------------- |
| DOUBLE  | 8 字节       | 近似计算 | 小（CPU 原生支持浮点运算） |
| FLOAT   | 4 字节       | 近似计算 | 小（CPU 原生支持浮点运算） |
| DECIMAL | -            | 精准计算 | 大                         |

建议在使用小数类型时，只指定数据类型，而不指定数据精度

需要使用小数存储和精准计算时，可以先将数据放大 X 倍，使用 BIGINT 存储，计算完成后，将结果再缩小 X 倍



##### 4.1.1.2 字符串类型

**【VARCHAR | CHAR】**

|              | VARCHAR                                                      | CHAR                                                |
| ------------ | ------------------------------------------------------------ | --------------------------------------------------- |
| **存储特点** | 1. 存储 **变长** 字符串 2. 需要额外使用额外字节记录字符串长度（未超过255字节为1，超过为2） | 1. 存储 **定长** 字符串 2. 会删除存储内容尾部的空格 |
| **优点**     | 仅使用必要的存储空间                                         | 不容易产生碎片                                      |
| **缺点**     | 1. 容易产生碎片 2. 更新变得更长时可能会导致页分裂            | 更新时无法扩展字段长度                              |
| **使用场景** | 1. 列更新频率低 2. 列使用了 UTF-8 之类的复杂字符集（每个字符使用不同字节数存储） | 列存储长度短且固定（MD5 值）                        |

**【BLOB | TEXT】**

BLOB 和 TEXT 都是为存储很大的数据而设计的字符串数据类型

- BLOB：采用二进制方式存储，没有排序规则和字符集
- TEXT：采用字符方式存储

MySQL 会将它们作为一个独立的对象处理，当它们过大时，InnoDB 会使用外部存储区域来存储实际数据值，内部只保留一个 1~4 字节的指针

==**问题**==：使用 ORDER BY 命令时，会扫描整个表，并将结果存储在临时表中（见 EXPLAIN 的 Extra 列包含 Using temporary），如果目标是占用空间较大的列（如 BLOB | TEXT | VARCHAR(1000) 等），就会使得临时表占用多空间

==**解决**==：使用 ORDER BY SUBSTRING (column, length) 命令将待比较的列值转换为截取字符串，有效减小临时表的大小



##### 4.1.1.3 日期和时间类型

|              | DATETIME            | TIMESTAMP（时间戳）                                          |
| ------------ | ------------------- | ------------------------------------------------------------ |
| **时间范围** | 1001年 — 9999年     | 1970年 — 2038年                                              |
| **存储空间** | 8 字节              | 4 字节                                                       |
| **时区影响** | 无                  | 有（与MySQL 服务器、操作系统、客户端连接的时区有关）         |
| **显示格式** | yyyy-mm-dd hh:mm:ss | 1970年1月1日0点0分0秒至今的秒数 MySQL 内部提供了 FROM_UNIXTIME() 函数将时间戳转换为日期 |



#### 4.1.2 Schema 的设计

##### 增加冗余字段和冗余表

增加冗余字段和冗余表的本质就是反范式化

优点：这样可以有效的减少关联查询，同时利用索引，优化读操作。

缺点：在进行写操作时，字段和表的维护变得更加困难

##### 避免 ALERT TABLE 的操作

MySQL 的 ALERT TABLE 操作非常影响性能，因为该操作会锁住整个表，具体过程如下：

1. 用修改后的新结构创建一个新的空表
2. 从旧表中查出所有数据插入新表
3. 最后删除旧表

解决方法：

- 主库调换法：先在一台不提供服务的机器上执行 ALERT TABLE 操作，然后再和主库替换
- 影子拷贝法：
  - 在主服务器上建立新的表，新表结构就是要修改老表之后的结构，然后把老表数据导入新表
  - 同时在老表建立一系列触发器，把老表的数据修改同步到新表
  - 当老表和新表的数据同步后再对老表建立一个排它锁，然后重新命名新表为老表的名字，最后删除后命名的老表。
  - 这样做的好处可以减少主从延迟，并且可以借助 pt-onlinee-schema-change 工具完成操作



### 4.2 索引优化

#### 概念

索引的目的是**提高查询效率**，本质上是一种**排好序**的**数据结构**

- B Tree 结构
  - m 叉 B 树每个节点最多有 m-1 个元素
  - 所有索引元素无重复
  - 叶节点之间没有指针

[![image-BTree](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/BTree.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/BTree.png)

- B+ Tree 结构 (默认)
  - m 叉 B+ 树每个节点最多有 m 个元素
  - 非叶子节点**不存储 Data，只存储索引**，因此可以放更多的索引
  - 叶子节点包含**所有索引字段**
  - 叶子节点用指针连接，提高区间访问性能

[![image-B+Tree](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/B%2BTree.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/B%2BTree.png)

- Hash 结构
  - 对索引的 key 进行一次 hash 计算就可以定位 Data 存储位置
  - 就单一查询来说，性能优于 B+ Tree
  - 无法进行范围查找，且存在 hash 冲突问题

[![image-Hash](https://github.com/IvanBao97/Notes/raw/main/MySQL/NotePics/Hash.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/Hash.png)

#### 优势和劣势

优势

- 提高数据检索效率，减少全表扫描次数
- 利用索引结构特性，降低数据排序成本

劣势

- 索引本质上也是一张表，需要占用额外的空间
- 索引虽然提高了查询速度，但会严重影响 INSERT, UPDATE, DELETE 速度。因为更新表时，MySQL 不仅需要保存数据，还需要更新索引指向

#### 索引分类

【**聚簇索引**】

- 将数据存储与索引放到了一块，找到索引也就找到了数据

【**非聚簇索引**】

- 将数据存储于索引分开结构，索引结构的叶子节点指向了数据的对应行

【**倒排索引**】

> 上述两种索引都是 B+Tree 结构，都可以通过索引字段的前缀进行查询
>
> SELECT * FROM tables WHERE name = 'xxx%'
>
> 但是，如果需要进行全模糊查询，索引就不会生效了
>
> SELECT * FROM tables WHERE name = '%xxx%'
>
> 搜索引擎中的全文检索就是这样的场景，使用了倒排索代替 B+Tree 索引





#### 如何使用索引

- 使用语法

  ```mysql
  CREATE INDEX idxName ON tableName(colName1, colName2, ...)
  ```

- 适合建立索引的情况

  - **主键自动建立唯一索引**
  - 在经常需要搜索的列上，可以加快搜索的速度
  - 在经常用在连接（JOIN）的列上，这些列主要是一外键，可以加快连接的速度
  - 在经常需要根据范围（<，<=，=，>，>=，BETWEEN，IN）进行搜索的列上创建索引，因为索引已经排序，其指定的范围是连续的
  - 在经常需要排序（order by）的列上创建索引，因为索引已经排序，这样查询可以利用索引的排序，加快排序查询时间；

- 不适合建立索引的情况

  - 表记录过少（MySQL 会默认全表扫描）
  - 表的字段需要经常被修改
  - 表的字段数据重复且分布均匀（例如：性别字段，只有男女两个数据且分布均匀）
  - 对于数据较大字段也不应该建立索引，如 text,  bolb



### 4.3 查询性能优化

#### Explain 参数介绍

EXPLAIN 命令是用来查看优化器决定的执行查询流程。

使用EXPLAIN 命令时，MYSQL 会在查询上设置一个标记，这个标记会使其返回关于在执行计划中每一步的信息，从而可以从分析结果中找到查询语句或是表结构的性能瓶颈。

EXPLAIN 不考虑各种 Cache 和显示 MySQL 在执行查询时所作的优化工作

```mysql
mysql> EXPLAIN SELECT * FROM blog_blog;+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------+| id | select_type | table     | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------+|  1 | SIMPLE      | blog_blog | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 5986 |   100.00 | NULL  |+----+-------------+-----------+------------+------+---------------+------+---------+------+------+----------+-------+
```

**【参数详解】**

|      **id**       | 代表执行 SELECT 子句或操作表的顺序（id相同，从上至下。id不同，从小到大） |
| :---------------: | :----------------------------------------------------------- |
|  **select_type**  | SELECT类型，可以为以下任何一种:<br/>**SIMPLE**：简单 SELECT (不使用 UNION 或子查询)<br/>**PRIMARY**：主查询，即最外面的查询<br/>**UNION**：UNION 中的第二个或后面的查询语句<br/>**DEPENDENT UNION**：UNION 中的第二个或后面的 SELECT 语句，取决于外面的查询<br/>**UNION RESULT**：UNION 的结果<br/>**SUBQUERY**：子查询中的第一个 SELECT<br/>**DEPENDENT SUBQUERY**：子查询中的第一个 SELECT，取决于外面的查询<br/>**DERIVED**：导出表的 SELECT (FROM 子句的子查询) |
|     **table**     | 输出的行所引用的表                                           |
|     **type**      | 联接类型。下面给出各种联接类型，按照从**最佳类型到最坏类型（由上到下性能逐渐变差）**进行排序:<br/>**system**：表仅有一行(=系统表)。这是 const 联接类型的一个特例。<br/><br/>**const**：表最多有一个匹配行，它将在查询开始时被读取。因为仅有一行，在这行的列值可被优化器剩余部分认为是常数。常见一个主键放置到 where 后面作为条件查询<br/><br/>**eq_ref**：类似ref，区别就在使用的索引是唯一索引，当找到目标数据后，就停止了查询，多表连接中使用 unique index 或者 primary key作为关联条件。<br/><br/>**ref**：对查找条件列使用了索引而且不为主键和 unique。其实，意思就是虽然使用了索引，但该索引列的值并不唯一，有重复。这样即使使用索引快速查找到了第一条数据，仍然不能停止，要进行目标值附近的小范围扫描。但它的好处是它并不需要扫全表，因为索引是有序的，即便有重复值，也是在一个非常小的范围内扫描。<br/><br/>**ref_or_null**：该联接类型如同ref，但是添加了 MySQL 可以专门搜索包含NULL值的行。<br/><br/>**index_merge**：该联接类型表示使用了索引合并优化方法。<br/><br/>**unique_subquery**：该类型替换了下面形式的IN子查询的 ref: value IN (SELECT primary_key FROM single_table WHERE some_expr) unique_subquery 是一个索引查找函数,可以完全替换子查询,效率更高。<br/><br/>**index_subquery**：该联接类型类似于 unique_subquery。可以替换IN子查询，但只适合下列形式的子查询中的非唯一索引: value IN (SELECT key_column FROM single_table WHERE some_expr)<br/><br/>**range**：只检索给定范围的行，使用一个索引来选择行，常见于**<，<=，>，>=，between**等操作符。<br/><br/>**index**：该联接类型与ALL相同，除了只有索引树被扫描，且扫描顺序是按照索引的顺序。这通常比ALL快，因为索引文件通常比数据文件小。<br/><br/>**ALL**：对于每个来自于先前的表的行组合，进行完整的表扫描，全表扫描。 |
| **possible_keys** | 指出 MySQL 能使用哪个索引在该表中找到行，表示查询时可能使用的索引。 |
|      **key**      | 显示 MySQL 实际决定使用的键(索引)。如果没有选择索引，键是 NULL。 |
|    **key_len**    | 显示 MySQL 决定使用的键长度。如果键是NULL，则长度为 NULL。   |
|      **ref**      | 显示使用哪个列或常数与 key 一起从表中选择行。                |
|     **rows**      | 显示 MySQL 认为它执行查询时必须检查的行数。多行之间的数据相乘可以估算要处理的行数。 |
|   **filtered**    | 显示了通过条件过滤出的行数的百分比估计值。                   |
|     **Extra**     | 该列包含 MySQL 解决查询的详细信息：<br>**Distinct**：MySQL 发现第1个匹配行后，停止为当前的行组合搜索更多的行。<br>**Not exists**：MySQL 能够对查询进行 LEFT JOIN 优化，发现1个匹配 LEFT JOIN 标准的行后，不再为前面的的行组合在该表内检查更多的行。<br/>**range checked for each record (index map: #)**：MySQL 没有发现好的可以使用的索引，但发现如果来自前面的表的列值已知，可能部分索引可以使用。<br/>**Using filesort**：说明 MySQL 会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取,mysql中无法利用索引完成的排序操作称为"文件排序"。<br/>**Using index**：说明使用了覆盖索引（Covering Index），避免访问了表的数据行，效率不错。如果同时出现 using where，表明索引被用来执行索引键值的查找；如果没有同时出现using where，表明索引用来读取数据而非执行查找动作。<br/>**Using temporary**：使用了临时表保存中间结果，MySQL 在对查询结果排序时使用临时表，常见于 order by 和 group by。<br/>**Using where**：表明使用了 WHERE 过滤。<br/>**Using index for group-by**：类似于访问表的 Using index 方式，Using index for group-by 表示 MySQL 发现了一个索引，可以用来查询GROUP BY 或 DISTINCT 查询的所有列，而不要额外搜索硬盘访问实际的表。 |



#### 优化查询语句结构

**【优化数据访问】**

* 减少请求不需要的数据
  * 有些查询会请求超过实际需要的数据，这会给 MySQL 服务器带来额外的负担，增加网络开销。在实际开发中要尽量避免下列类似查询发生：
    1. 查询不需要的记录
    2. 多表关联时返回全部列
    3. 重复查询相同的数据

* 减少扫描额外的数据行
  * MySQL查找数据时，会扫描数据行直至找到目标数据。在实际开发中，我们可以通过以下方式减少这种扫描成本，更快地找到目标数据
    1. 使用索引覆盖，把需要的列放在索引中，避免了回表操作
    2. 改变表结构，对于常用的跨表字段冗余到一张汇总表
    3. 重构查询方式，使 MySQL 优化器能更好地执行查询



**【重构查询方式】**

* 切分查询：将一次性大批量的查询分解为多批查询，减轻服务器的压力，也能减少 MySQL 复制的延迟。

```mysql
# employee 表中数据量很大，一次性查询将会增加服务器压力

-- 原始查询
SELECT * FROM employee;
-- 修改后
SELECT * FROM employee WHERE id >= 0 AND id < 1000;
SELECT * FROM employee WHERE id >= 1000 AND id < 2000;
...
```

* 分解关联查询：将关联查询以关联字段分解，成为多个单表查询，将大大提升查询效率。

```mysql
# employee 表和 department 表通过 employee_id 关联

-- 原始查询
SELECT 
	employee.name, department.address 
FROM 
	employee JOIN department ON employee.id = department.empid 
WHERE 
	employee.tel = 123xxxxxxxx; 

-- 修改后
SELECT employee.id AS targetId, employee.name FROM employee WHERE employee.tel = 123xxxxxxxx;
SELECT department.address FROM department WHERE department.empid = targetId;
```



#### 查询执行过程

[![image-B+Tree](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/MySQL_Query_Process.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/MySQL_Query_Process.png)

1. 客户端向 MySQL 服务器发出请求
   * MySQL 服务器和客户端间的通信是半双工的，不能双向同步数据，任意时刻只能由一方传输，一旦一端开始发送消息，另一端需要接收完消息才能响应。在 MySQL 服务端返回查询数据时，需要等所有的数据都发送给客户端才能够释放这条查询所占用的资源。当我们需要进行大量的数据查询时，例如需要查询几万条或几十万条运单的商户信息，请只返回我们真正需要的字段，尽量避免 `SELECT *`，这样能够减少数据传输的开销，减轻服务器和客户端的压力。
2. 服务器检查缓存（若 MySQL 开启了查询缓存），若存在缓存直接返回
   * MySQL 会把查询语句进行 Hash，作为缓存标识，如果 Hash 值存在，则说明有缓存，直接返回。
   * 如果表的结构或数据发生改变时，那么使用这个表的所有缓存查询将不再有效
   * ==注意==：MySQL 的缓存并不是什么场景都是好的，需要衡量缓存使用的开销和它能够给我们带来的收益。MySQL 服务器缓存是默认关闭的，因为在缓存的设置，删除以及更新都需要比较多的系统开销，综合收益并不大，另外在客户端层 Mybatis 的一级、二级缓存提供了非常相似的缓存功能，个人感觉还是在客户端层进行缓存会更好些。
3. 服务器解析 sql 语句，进行预处理，并由优化器生成相应的执行计划
4. MySQL 根据执行计划，调用存储引擎 API 来执行查询。
5. 将查询结果同步到缓存中（若 MySQL 开启了查询缓存）
6. 返回查询结果给客户端，并缓存查询结果。



#### 关联查询优化

**【实现原理】**

MySQL在处理连接时是以 **嵌套循环连接算法** 的方式来实现的

```python
for each row in table1{
	for each row in table2{
		for each row in table3{
		# 匹配判断
		...
		}
	}
}
```

[![join](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/Join.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/Join.png)



**【执行过程】**

多表连接时，MySQL 会先从驱动表 table1 中进行单表查询得到结果集 res1，然后拿 res1 结果集中的每条数据挨个作为条件去和被驱动表 table2 中进行单表查询得到的结果集 res2 进行匹配，最终得到匹配结果 res12，然后使用 res12 结果集中的每条数据挨个作为条件去和被驱动表 table3 中进行单表查询得到的结果集 res3 进行匹配，得到最终结果集 res123 并返回。（阿里巴巴开发手册中要求  JOIN 不能超过3个）

```mysql
SELECT
	col,
	col2,
	col3...
FROM
	TABLE A
    LEFT | RIGHT | INNER JOIN TABLE B ON < JOIN CONDITION > 
WHERE
	< WHERE CONDITION >;
```

1. FROM
   * SQL 语句是从 from 开始执行的，from 是对左右两张表进行笛卡尔积，产生第一张虚拟表 vt1。如果左表中的记录行数是 n，右表中的记录行数是 m，那么笛卡尔积产生的虚拟表中的记录行数为 n * m。
2. ON
   * 根据 on 的条件对 vt1 表进行筛选，将筛选后的结果保存到虚拟表 vt2。
3. JOIN
   * 这一步主要是添加外部行，
     * 如果是左连接 left join on，那么会先遍历左表中的每一行，然后不在 vt2 表中的记录将会被插入到 vt2，其余字段会置为 null，形成虚拟表 vt3。
     * 如果是右连接 right join on，那么会先遍历右表中的每一行，然后不在 vt2 表中的记录将会插入到 vt2，其余字段会置为 null，形成虚拟表 vt3。
     * 如果是内连接 inner join on 的话，则不会添加外部行。所产生的 vt3 表和 vt2 表是完全相同的。
4. WHERE
   * 对 vt3 表进行条件过滤，将筛选过滤后的结果保存到表 vt4。
5. SELECT
   * 按照查询的字段，从 vt4 表中取出所需字段，输出到 vt5 表中，那么最终的返回结果就是 vt5 表。

==参考==：[JOIN 中是否存在 WHERE 语句，具体过程详见此博客](https://www.cnblogs.com/shengdimaya/p/7123069.html)



**【优化方案】**

* 使用索引
  * 连接查询是使用嵌套循环连接算法来实现的，驱动表只访问一次，被驱动表需要被访问多次，在没有使用索引的情况下，驱动表和被驱动表每次单表查询都会是全表扫描，如果在一般情况下每次驱动表和被驱动表的单表查询是都使用到索引，查询效率就会大幅提高。
* 基于块的嵌套循环连接
  * 上面的基本嵌套循环连接查询和使用索引都是被驱动表中得到的结果集每次只和一条驱动表中记录进行匹配，那么在这个过程中驱动表中有多少条数据，就要访问多少次被驱动表进行单表查询，访问被驱动表就是要将被驱动表中的记录从硬盘加载到内存中，然后在内存中和驱动表中的数据进行匹配，然后将被驱动表中的记录从内存中移除，然后再从驱动表中拿出下一条记录重复这个步骤，由此可见这种方式会带来大量的IO操作，所以MySQL的设计者就提出了基于块的嵌套循环连接来减少被驱动表的访问次数，减少对应的 IO 操作。
  * 基于块的嵌套循环连接引入了 “join buffer” 概念，即在进行连接查询前申请一块固定大小的内存，将驱动表中结果集中若干条记录存储在 join buffer 中，然后扫描被驱动表，将被驱动中的每一条记录一次性和 join buffer 中的多条记录进行比较匹配，从而减少被驱动表的访问次数。



#### 优化器常见的优化

* **索引合并优化**
  * 当 WHERE 子句包含多个复杂条件时，MySQL 可以访问单个表的多个索引以合并和交叉过滤的方式来定位目标数据
* **列表 IN() 优化**
  * MySQL 会将 IN() 列表中的数据先进行排序，再通过二分查找的方式确定目标值
* **常量优化**
  * MySQL 会先检测查询语句中是否有表达式可以转化为常量，如果可以，则直接使用常量替代
  * 如果 MIN()、MAX() 处理的字段有索引，也会被替换（B+TREE 结构的最左和最右）



#### 特定查询语句的优化

* **优化 GROUP BY 和 ORDER BY 查询**
  * 确保 GROUP BY 和 ORDER BY 的表达式中只涉及一个表中的列，这样 MySQL 就可以利用索引优化查询过程，否则就会遍历数据，放在临时表中

* **优化 LIMIT 分页查询**

```MYSQL
# 当 LIMIT 偏移量较大时
SELECT * FROM empolyee ORDER BY cratetime LIMIT 1000,20;
# MySQL 需要先查询10020条数据，再抛弃前面10000条，返回最后20条，这样代价非常高
# 优化方法：

# 1. 使用覆盖索引 + 关联查询
SELECT empolyee.name 
FROM empolyee JOIN (
    SELECT empolyee.id AS id
    FROM empolyee 
    ORDER BY cratetime 
    LIMIT 1000,20;) AS lim ON empolyee.id = lim.id;

# 2. 在业务合理的情况下，使用 id 来进行分页（id 随着 createtime 增长）
SELECT * FROM empolyee ORDER BY id LIMIT 1000,20;
```

**【优化 UNION 查询】**

```MYSQL
# MySQL 无法把限制条件由外层推至内层
# 因此下列语句会在临时表中将两个子查询的所有结果联合，再取20条记录
(SELECT name FROM employee ORDER BY createtime)
UNION ALL 
(SELECT name FROM employer ORDER BY createtime)
LIMIT 20;
# 为了优化查询，我们需要在子查询中也加入限制条件
(SELECT name FROM employee ORDER BY createtime LIMIT 20)
UNION ALL 
(SELECT name FROM employer ORDER BY createtime LIMIT 20)
LIMIT 20;
# 注意，从临时表中取数据时，并不是顺序的，如果需要保证顺序，还需要加一个全局的 ORDER BY 在全局的 LIMIT 前面
```



### 4.4 分库分表

#### 分库分表的背景

起初，只有单机数据库工作，随着请求越来越多，将数据库的写操作和读操作进行分离， 使用多个从库副本（Slaver Replication）负责读，使用主库（Master）负责写， 从库从主库同步更新数据，保持数据一致。架构上就是数据库**主从同步**。 





后来，用户量级变多了，写请求越来越多，再加一个 Master 不能解决问题，这时就需要用到分库分表（Sharding），对写操作进行切分。



#### 分库分表的方式

**【垂直拆分】**

* 垂直分表
  * 将一个表按照字段分成多表，每个表存储其中一部分字段。一般是表中的字段较多，将不常用的， 数据较大，长度较长（比如text类型字段）的拆分到“扩展表“。避免 IO 效率低的问题（原因如下）
    * 第一是由于数据量本身大，需要更长的读取时间；
    * 第二是跨页，页是数据库存储单位，很多查找及定位操作都是以页为单位，单页内的数据行越多数据库整体性能越好，而大字段占用空间大，单页内存储行数少，因此IO效率较低。
    * 第三，数据库以行为单位将数据加载到内存中，这样表中字段长度较短且访问频率较高，内存能加载更多的数据，命中率更高，减少了磁盘 IO，从而提升了数据库性能。

[![垂直分表](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/%E5%9E%82%E7%9B%B4%E5%88%86%E8%A1%A8.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/%E5%9E%82%E7%9B%B4%E5%88%86%E8%A1%A8.png)

* 垂直分库
  * 垂直分库是指按照业务将表进行分类，分布到不同的数据库上面，每个库可以放在不同的服务器上，它的核心理念是专库专用。

[![垂直分库](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/%E5%9E%82%E7%9B%B4%E5%88%86%E5%BA%93.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/%E5%9E%82%E7%9B%B4%E5%88%86%E5%BA%93.png)

**【水平拆分】**

* 水平分表
  * 针对数据量巨大的单张表（比如订单表），按照某种规则（RANGE、HASH取模等），切分到多张表里面去。 但是这些表还是在同一个库中，所以库级别的数据库操作还是有 IO 瓶颈，仅作为水平分库的一个补充优化。

[![水平分表](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/%E6%B0%B4%E5%B9%B3%E5%88%86%E8%A1%A8.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/%E6%B0%B4%E5%B9%B3%E5%88%86%E8%A1%A8.png)

* 水平分库
  * 可以把一个表的数据 (按数据行) 分到多个不同的库，每个库只有这个表的部分数据，这些库可以分布在不同服务器，从而使访问压力被多服务器负载，大大提升性能。它不仅需要解决跨库带来的所有复杂问题，还要解决数据路由的问题。

[![水平分库](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/%E6%B0%B4%E5%B9%B3%E5%88%86%E5%BA%93.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/%E6%B0%B4%E5%B9%B3%E5%88%86%E5%BA%93.png)

**【水平切分规则】**

* RANGE
  * 从0到10000一个表，10001到20000一个表
* HASH 取模
  * 一个商场系统，一般都是将用户，订单作为主表，然后将和它们相关的作为附表，这样不会造成跨库事务之类的问题。 取用户id，然后hash取模，分配到不同的数据库上。



#### 分库分表后面临的问题

**【全局主键避重问题】**

我们往往直接使用数据库自增特性来生成主键ID，这样确实比较简单。而在分库分表的环境中，数据分布在不同的分片上，不能再借助数据库自增长特性直接生成，否则会造成不同分片上的数据表主键会重复。简单介绍几种ID生成算法。

1. Twitter 的 Snowflake（64位唯一 Id，由41位的 timestamp + 10位自定义的机器码 + 13位累加计数器组成）
2. UUID/GUID（一般应用程序和数据库均支持）
3. MongoDB ObjectID（类似UUID的方式）
4. Ticket Server（数据库生存方式，Flickr采用的就是这种方式）



**【跨库 Join】**

拆分后，数据库可能是分布式在不同实例和不同的主机上，JOIN 将变得非常麻烦。而且基于架构规范，性能，安全性等方面考虑，一般是禁止跨库 JOIN 的，通常可以采用下列方式解决：

1. 全局表
   * 所谓全局表，就是有可能系统中所有模块都可能会依赖到的一些表。比较类似我们理解的“数据字典”。为了避免跨库join查询，我们可以将这类表在其他每个数据库中均保存一份。同时，这类数据通常也很少发生修改（甚至几乎不会），所以也不用太担心“一致性”问题。
2. 字段冗余
   * 这是一种典型的反范式设计，在互联网行业中比较常见，通常是为了性能来避免 join 查询。
3. 数据同步
   * A 库中的 tab_a 表和 B 库中 tbl_b 有关联，可以定时将指定的表做同步。当然，同步本来会对数据库带来一定的影响，需要性能影响和数据时效性中取得一个平衡。这样来避免复杂的跨库查询。
4. 系统层组装
   * 在系统层面，通过调用不同模块的组件或者服务，获取到数据并进行字段拼装。组装的时候要避免循环调用服务，循环RPC，循环查询数据库，最好一次性返回所有信息，在代码里做组装。



**【 Order by | Group By | Limit | 聚合函数问题】**

多库进行查询时，可能会出现跨节点分页或排序的问题。需要在两个节点上先分别操作（分页、排序），然后合并数据之后，再重新操作（分页、排序）。

max、sum、count 之类的函数在进行计算的时候，也需要先在每个分片上执行相应的函数，然后将各个分片的结果集进行汇总和再次计算，最终将结果返回。



**【事务支持】**

详细介绍见 **4.5 MySQL 其他特性**



### 4.5 MySQL 其他特性

#### 分区表





#### 字符集

MySQL 通常可以在两个时期设置字符集：

* 创建数据对象时设置
  * 创建数据库时，如果没有显示设置 `CHARSET = xxx`，就会读取配置文件中 character_set_server 字段的值指定数据库的字符集

```mysql
CREATE TABLE `user` (
  `id` bigint(20) NOT NULL,
  `name` varchar(20) NOT NULL DEFAULT '' COMMENT '名字',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表'

# 【utf8 和 utf8mb4 区别】：
# 1. utf8mb4 是 utf8 的超集，专门用来兼容四字节的 unicode。当然，为了节省空间，一般情况下使用 utf8 也就够了。
# 2. mysql支持的 utf8 编码最大字符长度为 3 字节，如果遇到 4 字节的宽字符就会插入异常了（包括 Emoji 表情，和很多不常用的汉字）
```

* 服务器和客户端通信时设置

[![CharSet](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/charset.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/charset.png)



#### 分布式事务

当对数据库采用垂直拆分和水平数据分片时，会将数据拆分到多个不同的数据节点上，如果一个事务里的操作涉及了多个不同分片节点，为了保证不同分片节点上数据的一致性，就产生了分布式事务。

**【分布式事务实现方式（其二）】**

一、两阶段提交（2PC 模式）

* 分布式事务通常采用 2PC 协议，全称 Two Phase Commitment Protocol。分布式事务通过 2PC 协议将提交分成两个阶段：
  * 阶段一为准备（prepare）阶段。即所有的参与者准备执行事务并锁住需要的资源。参与者 ready 时，向 transaction manager 报告已准备就绪。
  * 阶段二为提交阶段（commit）。当 transaction manager 确认所有参与者都 ready 后，向所有参与者发送 commit 命令。
* 事务协调者 Transaction Manager
  * 因为 XA 事务是基于两阶段提交协议的，所以需要有一个事务协调者（transaction manager）来保证所有的事务参与者都完成了准备工作(第一阶段)。如果事务协调者（transaction manager）收到所有参与者都准备好的消息，就会通知所有的事务都可以提交了（第二阶段）。

[![2pc](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/2pc.png)](https://github.com/IvanBao97/Notes/blob/main/MySQL/NotePics/2pc.png)

* 缺点
  * 同步阻塞。执行过程中，所有参与节点都是事务阻塞型的。当参与者占有资源时，其他第三方节点访问资源不得不处于阻塞状态。
  * 单点故障。由于协调者的重要性，一旦协调者发生故障。参与者会一直阻塞下去。尤其在第二阶段，协调者发生故障，那么所有的参与者还都处于锁定事务资源的状态中，而无法继续完成事务操作。
  * 性能十分低下。2PC 中协调者与每个参与者至少有 2 轮消息交互、多次写日志，过程又是同步阻塞。

二、基于本地消息表

核心思想将分布式事务分成多个本地事务，这里称之为主事务与从事务。主事务本地先行提交，然后通过消息通知从事务，从事务从消息中获取信息进行本地提交。可以看出这是一种异步事务机制、只能保证最终一致性；但可用性非常高，不会因为故障而发生阻塞。

同时，通过一个“消费状态表”来记录消费状态，避免消息被重复消费。在执行“加款”操作之前，检测下该消息（提供标识）是否已经消费过，消费完成后，通过本地事务控制来更新这个“消费状态表”。

缺点：

* 关系型数据库的吞吐量和性能方面存在瓶颈，频繁的读写消息会给数据库造成压力。在真正的高并发场景下，该方案也会有瓶颈和限制的。

==参考==：[分布式事务最经典的七种解决方案](https://segmentfault.com/a/1190000040321750)



**【MySQL 的 XA 事务分为外部 XA 和内部 XA】 **

外部 XA：用于跨多MySQL实例的分布式事务，需要应用层作为协调者，通俗的说就是比如我们在PHP中写代码，那么PHP书写的逻辑就是协调者。应用层负责决定提交还是回滚，崩溃时的悬挂事务。MySQL数据库外部XA可以用在分布式数据库代理层，实现对MySQL数据库的分布式事务支持，例如开源的代理工具：网易的DDB，淘宝的TDDL等等。


内部 XA：事务用于同一实例下跨多引擎事务，由Binlog作为协调者，比如在一个存储引擎提交时，需要将提交信息写入二进制日志，这就是一个分布式内部XA事务，只不过二进制日志的参与者是MySQL本身。Binlog作为内部XA的协调者，在binlog中出现的内部xid，在crash recover时，由binlog负责提交。(这是因为，binlog不进行prepare，只进行commit，因此在binlog中出现的内部xid，一定能够保证其在底层各存储引擎中已经完成prepare)。



#### 查询缓存

**【MySQL 如何判断缓存命中】**

MySQL 会将你的查询语句（数据库和客户端协议版本）进行 Hash 处理，将 Hash 值和查询结果存在放在引用表中。当下一次查询到来时，先比对引用表中的Hash值，如果存在，则命中缓存，不再需要解析查询语句和查询数据库，直接返回缓存结果。

下列查询无法进行缓存：

* 查询语句中包含不确定函数，如 CURRENT_DATE、NOW
* 查询数据结果太大，超过缓存分配内存中的可用空间

**【缓存的优缺点】**

* 优点
  * 对于复杂查询确实能够提示查询效率，如关联查询后的 GROUP BY / ORDER BY
* 缺点
  * 表的写入操作会导致该表的缓存失效，这个过程是加锁的，如果缓冲较多，非常影响性能
  * 内存空间的碎片化，影响内存的空间利用率（虽然可以使用 FLUSH QUERY CACHE 命令整理碎片，但这个操作也会导致服务器僵死一段时间）

**【InnoDB 的查询缓存】**

因为 InnoDB 有自己的 MVCC，所以它的查询缓存更加严格。

* InnoDB 表的内存数据字段都保存了一个事务 ID 号，如果当前事务 ID 小于该表保存的事务 ID，则无法查询缓存
  * Eg：当前事务 ID 为5，且获得了锁，进行事务提交操作，那么事务1-4都无法再读取或写入该表的缓存
* 如果表上有任何的锁，那么任何查询语句都无法读取这个表的缓存



## 5. 复制






