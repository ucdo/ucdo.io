+++
date = '2025-06-25T09:22:52+08:00'
draft = false
title = 'Mysql_baguwen'
+++

 # MySQL8.0

- 主要是围绕存储和查询做文章。
- 学习的时候思考：
    - 怎么存储保证数据的完整性
    - 怎么存储保证数据的查询高效
    - 怎么查询保证数据的正确性
    - 怎么优化查询保证查询高效性

## 存储 -- 由存储引擎实现 -- 这里只讨论 innodb ![img_1.png](source/img_1.png)

### 引子

- 数据存在`.idb`文件中： `表数据`
- version < 8.0
    - `.frm` 表结构文件
    - `.opt` 默认字符集和字符集规则
- version >= 8.0
    - `.frm` 改成了存放在`innodb内部系统表`中

### 物理存储模型

- 表空间(tableSpace)
    - 段(segment) : 一般分为`数据段，索引段，回滚段`：
      索引段：B+树非叶子节点的区的集合；数据段：B+树叶子节点的区的集合；回滚段：存放回滚数据的区的集合，例如MVCC，利用回滚段实现多版本数据查询
        - 区(extent)  `1M` 包含 `64个连续页` :
            - 页(page) `16kb` ： 读取的基本单位
                - 行(row) 由 `Dynamic存储`: 存储的基本单位：
                  变长字段实际长度，NULL值列表（类似bitmap，那个字段为null用0，1控制），
                  记录头信息（delete_mask，next_record,record_type），
                  row_id(没有主键约束或者唯一约束时存在),trx_id(事务id，由哪个事务创建，这是时必须要有的),roll_ptr(
                  记录上一个版本的指针),实际数据
                  。如果一行存储的内容大于 `16k` 也就是大于一页的存储空间，则会溢出页的指针，因为数据被存放在溢出页中了
                - ![img.png](source/img.png)

### 索引的存储 ： 主键索引和辅助索引是分开存放的

#### 主键索引（聚簇索引） 只能有一个

以B+树的方式存储，只在叶子节点存储实际数据

#### 辅助索引（二级索引） 可以有多个

也是以B+树的方式存储，只不过在叶子节点存的是，主键字段和索引字段

#### 所以为什么叫回表呢，就是因为要查询的数据并不全在辅助索引里面。

## 查询 ![img_2.png](source/img_2.png)

### 引子

- 执行一条`SQL`发生了什么
    - 建立连接，身份校验
    - 解析`SQL`
        - 词法，语法分析，构建语法树
    - 执行`SQL`
        - 预处理：检查表和字段是否存在；扩展 * 为对应表的所有字段
        - 优化：选择执行成本最小的方案
        - 执行：执行`SQL`查询语句，从存储引擎读取数据，返回给客户端

### 索引

#### 分类

1. 按数据结构分：`B+树索引`，`hash索引`，`full-text索引`
2. 按物理存储分：`聚簇索引（主键索引）`，`二级索引（辅助索引）`
3. 按字段特征分：`主键索引`，`唯一索引`，`普通索引`
4. 按字段个数分：`单列索引`，`联合索引`

#### 为什么选B+树作为索引存储的方式 （核心：树的高度会影响磁盘I/O的次数）

1. B+树相较于B树
    1. 只在叶子节点存储数据，可以存储更多的数据，所以高度会更低，所以磁盘I/O会变少，所以会更快
    2. B+树又很多冗余节点，插入删除都很高效，删除时可以直接从叶子节点删除。树形结构变化很小。
    3. B+树的叶子节点之间有双向链表连接，范围查询更方便。
2. B+树相较于二叉树，有更低的层级，因为它一个节点能存储多个数据，在数据量大的时候，进行的磁盘I/O更少
3. B+树相较于Hash，因为有双向链表，更适合做范围查询。而Hash在等值查询时非常快。

#### 创建索引

唯一索引：字段值不能重复，但是可以为空

```mysql
create unique index index_name on table_name (col1, col2);
```

普通索引

```mysql 
create index index_name on table_name (col1, col2);
```

#### 最左匹配原则

创建有一个联合索引 index_abc(a,b,c)  走索引的情况  
`
#因为有查询优化器的存在，所以下面的where子句的顺序不重要
where a = 1 and b  = 2 and c = 1;
where a  = 1;
where a = 1 and b = 1;
`  
不走索引，因为不满足联合索引最左匹配原则
`
where b = 2;
where b = 2 and c = 2;
where c = 2 ;
`

最左匹配原则是使用联合索引的关键

```mysql
create index index_a_b on t1 (a, b);
select *
from t1
where a > 1
  and b = 2;
select *
from t1
where a >= 1
  and b = 2;
select *
from t1
where a between 2 and 8
  and b = 2;
```

以上：a走索引，b=2被下推到了存储引擎层，因为它对a查询出来数据进行了过滤，减少了非 b = 2的数据回表，也很符合索引的价值。

#### 索引优化

1. 主键索引优化：主键索引设置为 `auto_increment`。避免插入和重排
2. 覆盖索引优化：把需要回表的字段添加到联合索引
3. 索引列设置为 `NOT NULL`:
    1. 减少存储空间，因为每行都有个字段用来记录NULL的bitmap
    2. 查询优化更直接：会忽略掉 `IS NULL` `IS NOT NULL` 的顾虑
    3. 避免内置函数（sum(),count(),join）对`NULL` 的特殊处理
    4. 联合索引优化：如果前面的列为 `NULL`会对后面的列造成影响
4. 索引失效
    1. 对索引字段进行模糊查询，如 `like '%xx'` 或者 `like '%xx%'` 。这里需要注意的是 `explain SQL`时，`type`可能是
       `index/ALL`这个要看有没有回表
    2. 在查询中对索引列做了计算、函数、类型转换操作
        1. 计算：select * from users where id + 1 = 2。[type = index || type = ALL]。回不回表看查询的字段
        2. 类型转换：比如name 是varchar类型，存储的值为22。使用select * from users where name =
           22；触发隐式类型转换。[type = index || type = ALL]。回不回表看查询的字段
        3. 函数：select name from users where length(name) = 6; [type = index || type = ALL]。回不回表看查询的字段
    3. 使用联合索引未遵循最左匹配原则
    4. 在使用 `WHERE OR` 的时候，`OR`前面是索引列，后面不是索引列
        1. explain select id from user where id = 1 or email = 'xx' [type = ALL]
5. 执行计划
    1. type：扫描的类型（性能由低到高）：ALL index range ref eq_ref const
        1. ALL 全表扫描，性能巨差
        2. index 全索引扫描，稍微比ALL好一点，没有回表。但是可以通过调整SQL语句或者建立适当的索引来优化
        3. range 索引范围扫描，性能较好
        4. ref 非唯一键索引扫描，性能很好，效率高，可能会有多行匹配
        5. eq_ref 唯一键索引扫描，性能非常好，效率极高
        6. const 只查询一条走主键索引或者唯一键索引，最佳的，无需优化
    2. passible_keys : 可能使用到的索引
    3. key: 实际使用的索引
    4. key_len：使用索引的长度
    5. ref：关联字段
    6. row：扫描了多少行数据
    7. Extra：
        1. Using filesort: 在内存或者磁盘中进行了排序。性能影响巨大
        2. Using temporary：通常指使用临时表来进行复杂的Group By,Order by 。性能影响巨大
        3. Using index：覆盖索引，无需回表，好得很
        4. Using where: 回表后对数据进行过滤，最不希望同时伴随 type=ALL 或者 type = index
        5. Using index condition: 索引下推。讲where的部分子句下推到存储引擎层，减少了回表次数（之前讨论的 a > 1 AND b = 2）
        6. Using join buffer: 连接缓存

### 分页优化

1. 原来的：select * from user order by id desc limit 1234567,20; 这里会查询出来
2. explain 看一下他为了获得这20行付出了什么代价
3. [根据查询字段看是否回表] 全表/全索引扫描。扫描的行数 = 1234567 + 20。 为了这20条数据，会丢弃前面 1234567条数据
4. 子查询优化 select * from user where id >= (select id from user order by id desc limit 123456,1) order by id desc
   limit 20
5. 在对深度分页做优化时，需要建议一个二级索引。
6. 为什么子查询会优化深度分页？
    1. 如果原来走主键索引查询，会查询出来很多数据，因为他是聚簇索引。或者因为查询的是 * 需要回表。但是回表的代价很昂贵
    2. 如果用子查询就走了二级索引，查询的数据非常少。就只有20条需要回表查询

### 事务

基本特征：ACID

1. 原子性：对于同一个事务中的所有操作，要么全部成功，要么全部失败。 [通过undo log(回滚日志)保证]
2. 一致性：事务操作前后，数据保持完整约束性，数据库保持一致型。比如说转账，正确的扣减和增加。
3. 隔离性：数据库允许多个事务并发读取和修改数据的能力，隔离性保证多个事务交叉执行时数据的一致性。
4. 持久性：事务对数据的修改是永久的。[通过 redo log(重做日志)保证]

#### 事务并发时可能会出现什么问题？

1. 脏读：![img_4.png](source/img_4.png)
    1. 概念：**读到一个事务修改过但是还未提交的数据**,
    2. 实操：**事务B读取了事务A修改后的数据，如果事务A回滚，那么事务B拿到的就是过期数据，所以叫脏读**
2. 不可重复读：![img_5.png](source/img_5.png)
    1. 概念：**在一个事务内多次读取同一条数据，前后两次获取到的数据不一致**
    2. 实操：**事务A在两次读取的间隔中，数据被修改了**
    3. 打个比方：用户需要下单10件商品，第一次读取库存时，剩余10件，经过了复杂的计算，物流准备等，再次读取库存时，发现库存不足（比如只有9件），这时就没法下单，用户体验巨差。
       或者没有二次校验，导致超卖或者带锁扣减失败
3. 幻读：**这里要重点学习一下next-key lock ,gap lock**
    1. 看图说话：![img_3.png](source/img_3.png)
    2. 概念：**一个事务多次查询符合条件的记录数量，出现了前后两次查询出来不一致的情况**
    3. 实操：
    4. 解决办法：
        1. 普通的select 通过 MVCC 多版本并发控制，来解决，即事务开始时创建一个快照，读取就是在快照中读取
        2. select for update 通过加 next-key lock 来进锁定满足记录的行，阻塞锁住范围内的数据插入

#### 事务

`start/begin transaction` 开始事务，当有查询时才启动
`start/begin transaction with consistent snapshot` 开始事务并立即启动

#### 读视图/快照 + MVCC

- 针对innodb和可重复读
- 读视图是在创建事务后执行第一个非加锁的select是开启。有一个creator_trx_id
- m_ids 是创建读视图时活跃但未提交的事务ids
- min_trx_id 是m_ids里面最小的事务id
- max_trx_id 是下一个带创建的事务id
- 数据行有两个重要的隐藏列
    - trx_id : 对数据进行 新增，修改，删除时的事务id
    - roll_pointer：回滚指针，指向undo log中该数据的上一个版本

- 可见性的核心就是和m_ids进行比较
- trx_id < min_trx_id 可见 ： 因为数据在事务创建之前就已经提交
- trx_id > max_trx_id 不可见 ： 因为u数据在事务创建之后才提交
- min_trx_id <= trx_id < max_trx_ids
    - trx_id == creator_trx_id 可见 ： 因为时当前事务提交的
    - trx_id in m_ids 不可见 ： 创建时事务活跃，为了保证一致性，所以不可见
    - trx_id not int m_ids 可见 ： 该行在事务创建的时候已经提交了

#### 怎么避免幻读的问题 特指可重复读

- （快照读）MVCC 通过多版本控制机制，来保证读取数据的一致性。（就是事务执行过程中看到的数据和启动时看到的数据是一致的，即使中途有事务插入了一条数据，也是查询不出来的）
- （当前读） select for update 通过 next_key lock（记录锁+间隙锁）机制，在next-key lock 范围内，如果有其实事务插入数据，会被阻塞。以此来避免幻读

### 锁

#### 全局锁： `flush tables with read lock` `unlock tables`

1. 特征：开启之后，数据库就变成了只读
2. 使用场景：全数据库备份
3. 坏处：会影响 写操作，导致业务停滞
4. 优化：在使用导出工具 mysqldump的时候加上 -single-transaction，就会在备份之前开启事务。在可重复读的隔离级别下，事务开启然后执行查询之后就读快照，然后对不会影响其他操作

#### 表锁

1. read lock = 读锁 = 共享锁
    - 用法: lock tables user read;
    - 本线程
        - 只能对 user 表进行读操作，
        - 对 user 表进行写操作会报错提示：当前表已经上锁，没法写
        - 没法对其他表进行读操作
    - 其他线程
        - 可以对 user 表进行读操作
        - 对 user 表进行写操作会被阻塞
    - 释放： unlock tables: 会释放当前进程加的所有锁，退出会话的时候也会释放

2. write lock = 写锁 = 排他锁 = 独占锁
    - 用法： lock tables user write;
    - 本线程：只能进行写操作。读操作也会被阻塞
    - 其他线程：不能操作，读写都会被阻塞

#### 意向锁

意向锁的目的是为了让后来的加锁操作能够快速知道这张表里面的哪些记录是加了什么锁，而不用每条数据去遍历查看

#### 什么是 IX

#### 什么是 IS

#### 自增锁 id自增时的锁

#### 行锁 Innodb才支持行锁

1. 普通的select不会加行锁，是通过MVCC保持读一致的。其实普通读也会开启事务，只不过是隐式的。而且是自动提交。
2. select ... lock in share mode 。 对数据加上读锁（s锁），即当前事务可以进行对加锁数据读操作，其他事务也能对加锁数据进行读操作，但是都不能对对加锁数据行写操作
3. select ... for update 。 对数据加上写锁，本事务内可以进行读写操作，阻止其他事务对加锁数据的读写操作。
4. s锁和s锁兼容，即可以重复加s锁
5. x锁和s锁/x锁不兼容。加了x锁不能再加s锁或者x锁

##### 记录锁 record lock 记录锁有 s锁和x锁之分

- select id from user where id = 1 for update. 给id = 1这条数据加上 x锁。其他事务不能对它进行操作。（想想下单扣减库存）
- select id from user where id = 1 lock in share mode. 只读

##### 间隙锁 gap lock

- 只存在与 **可重复读** 隔离级别
- 目的是未了解决 可重复读 下的插入幻影的问题。即 id = 3,id = 5,gap lock锁住这个间隙，避免id = 4的数据被写入

#### next-key lock

- 锁定数据本身以及间隙。阻止其他事务插入到间隙中。
- 虽然多个事务之间的间隙锁是兼容的，但是要考虑到记录锁的冲突

#### mysql是怎么加行锁的 (select for update 的目的是锁住存在的记录，又要避免其他事务的写入造成幻读)

- 加锁的对象是索引，加锁的基本的基本单位是next-key lock。可能会退化成记录锁或者间隙锁

#### 乐观锁
通过version进行控制，典型的场景：库存扣减。
```mysql 
UPDATE inventory
SET
    stocks = CASE
                 WHEN id = 1 THEN stocks - 1
                 WHEN id = 2 THEN stocks - 2
                 ELSE stocks -- 明确地保留其他 ID 的 stock 值，防止意外变 NULL 或其他行为
             END,
    version = CASE
                  WHEN id = 1 THEN version + 1
                  WHEN id = 2 THEN version + 1
                  ELSE version -- 明确地保留其他 ID 的 version 值
              END
WHERE
    (id = 1 AND stocks >= 1 AND version = 21)
    OR
    (id = 2 AND stocks >= 2 AND version = 21);
```

#### 唯一索引等值查询

##### 值存在的情况下

next-key lock退化成 record lock。因为这条数据存在，又同时加上了 IX和X锁，其他事务没法对这条数据进行操作。所以能够保证数据的一致性。

##### 值不存在的情况下

- 值位于两条数据之间时：X意向锁，Gap lock

### 加锁查询 特别注意的是要防止全表扫描导致的表锁

select * from performance_schema.data_locks

#### 加锁的规则 （以下只针对innodb）

#### 唯一索引（主键索引）

`select * from user where id condition id_ for update`

##### id = 3的数据存在时

因为数据存在且等值查询，next-key lock 退化成 record lock。因为是等值查询，不需要锁间隙，但是需要防止被修改。

##### id = 3 的数据不存在时

因为不存在，next-key lock 所以退化成 gap lock。在目标值之间加上gap lock。防止被写入该值对应的数据，保持一致性。

##### 范围查询时：

唯一索引范围查询时，对满足条件的每一条数据加上记录锁，并且对经过的索引之间都加上间隙锁。避免存在的数据被修改，避免满足条件的间隙之间被写入数据。

#### 非唯一索引

##### 等值查询存在时： 对满足条件的记录加上 next-key lock.

##### 等值查询不存在时： 对目标值所在间隙之间加上 gap lock，防止被写入满足条件的数据

##### 范围查询：对满足条件的每一条记录和间隙之间加上锁，以此来防止幻读

## 日志

### WAL ： 数据库的写操作不是立刻写入到磁盘，而是先写日志，再在合适的时机写到磁盘上 （从随机写变成是了顺序写，提升了磁盘的写入性能）

### buffer pool： 提高读写效率，把数据页load进内存，修改和新增都发生在内存中，再通过合适的时机写入磁盘进行持久化

![img_7.png](source/img_7.png)

### redo log 持久化 ： 记录某个数据页修改了什么东西。比如 A表空间中的xx数据页中的zzz偏移量，做了yyy修改。 用于事务崩溃恢复

1. redo log + WAL : innodb可以保证数据库就算异常重启了，之前的记录也不会丢失，这种能力被称为 奔溃恢复。
2. 让随机写变成顺序写，提高了性能
3. 两个ib_logfile 循环写入![img_8.png](source/img_8.png)
4. 用作故障恢复

### undo log ：原子性 、 MVCC相关（版本链） ： 用于事务回滚

1. 原子性：利用事务回滚，保证事务的原子性：事务处理的过程中，如果出现了错误或者ROLLBACK，MySQL可以利用undo
   log中历史数据恢复到事务开始时的状态

2. MVCC 相关： MVCC = ReadView + undo log 实现：undo log会为每条记录保存多份历史版本，MySQL执行快照读的时候，事务会根据ReadView里面的
   信息，顺着undo log的版本，找到其可见的版本

### binlog 主从复制 全量记录