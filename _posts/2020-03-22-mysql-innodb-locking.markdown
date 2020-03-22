---
layout: post
title: "MySQL InnoDB锁（翻译自官方手册）"
author: "BJ大鹏"
header-style: text
tags:
  - MySQL
  - InnoDB
  - 锁
---

本文基于mysql 8.0，官方手册: <https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html>，同时参考了[mysql锁机制详解](https://www.cnblogs.com/volcano-liu/p/9890832.html)

主要内容如下：
1. 共享锁和排他锁(Shared and Exclusive Locks)
2. 意向锁(Intention Locks)
3. 记录锁(Record Locks)
4. 间隙锁(Gap Locks)
5. 邻键锁(Next-Key Locks)
6. 插入意向锁(Insert Intention Locks)
7. 自增锁(AUTO-INC Locks)
8. 空间索引的谓词锁(Predicate Locks for Spatial Indexes)

## 共享锁和排他锁(Shared and Exclusive Locks)
InnoDB实现标准的行级锁定，其中有两种类型的锁： 共享（S）锁和排他（X）锁。
- 共享（S）锁允许持有锁，读取行的事务。
- 独占（X）锁允许持有锁，更新或删除行的事务。

如果事务T1在r行持有S锁，则来自某些不同事务T2的对r行锁定的请求将按以下方式处理：
- T2请求S锁可以立即被授予。其结果是，T1与T2都在r上持有S锁。
- T2请求X锁不能立即授予。

如果事务T1在r行拥有独占（X）锁，则不能立即批准某个不同事务T2对r上任一类型的锁的请求。相反，事务T2必须等待事务T1释放对r行的锁定。

<mark><u>注：共享锁之间不互斥，简记为：读读可以并行。排他锁与任何锁互斥，简记为：写读，写写不可以并行。</u></mark>

## 意向锁(Intention Locks)
InnoDB支持多种粒度锁定，允许行锁和表锁并存。例如，<ins>LOCK TABLES ... WRITE</ins>这样的语句在特定表上采用排他锁（X锁）。为了使在多个粒度级别上的锁定变得切实可行，InnoDB使用意图锁。意向锁是表级锁，指示事务稍后对表中的行需要哪种类型的锁（共享锁或排他锁）。有两种类型的意图锁：
- 意图共享锁（IS）指示一个事务打算在表中每行设置一个共享锁。
- 意图独占锁（IX）指示一个事务打算在表中每行设置一个排他锁。

例如，<ins>SELECT ... FOR SHARE</ins>设置IS锁, <ins>SELECT ... FOR UPDATE</ins>设置IX锁.
意向锁定协议如下：
- 在事务可以获取表中某行的共享锁之前，它必须首先获取表中的IS锁或更高级别的锁。
- 在事务可以获取表中某行的排它锁之前，它必须首先获取该表的IX锁。

表级锁类型的兼容性汇总在以下矩阵中。

|  | X | IX | S | IS
---|---|---|---|---|---
X | 冲突 | 冲突 | 冲突 | 冲突
IX | 冲突 | <mark>兼容</mark> | 冲突 | <mark>兼容</mark>
S | 冲突 | 冲突 | <mark>兼容</mark> | <mark>兼容</mark>
IS | 冲突 | <mark>兼容</mark> | <mark>兼容</mark> | <mark>兼容</mark>

如果一个锁与现有锁兼容，则将其授予请求的事务，但如果与现有锁冲突，则不授予该锁。事务等待直到冲突的现有锁被释放。如果锁定请求与现有锁定发生冲突，不能被授予许可，因为可能导致死锁，发生错误。

意向锁不会阻止除全表请求（例如<ins>LOCK TABLES ... WRITE</ins>）以外的任何内容。意向锁的主要目的是表明有人正在锁定表中的行，或者打算锁定表中的行。

对于意图锁定事务数据出现类似于在下面<ins>SHOW ENGINE INNODB STATUS</ins>和 InnoDB的监视器输出：
```sql
TABLE LOCK table `test`.`t` trx id 10080 lock mode IX
```

## 记录锁(Record Locks)
记录锁是索引记录上的锁。例如，```SELECT c1 FROM t WHERE c1 = 10 FOR UPDATE;``` 防止任何其他事务插入、更新或删除 t.c1值为10的行。

记录锁始终锁定索引记录，即使定义的表没有索引。对于这种情况，InnoDB 创建一个隐藏的聚集索引并使用该索引进行记录锁定。

记录锁的交易数据类似于 <ins>SHOW ENGINE INNODB STATUS</ins> 和 INNODB 监视器输出中的以下内容:
```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10078 lock_mode X locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
2: len 7; hex b60000019d0110; asc        ;;
```

## 间隙锁(Gap Locks)
间隙锁是对索引记录之间的间隙的锁，或者是对第一个索引记录之前或最后一个索引记录之后的间隙的锁。例如，```SELECT c1 FROM t WHERE c1 BETWEEN 10 and 20 For UPDATE;``` 阻止其他事务将值15插入到列 t.c1中，无论列中是否已经有这样的值，因为范围中所有现有值之间的间隙都被锁定。

间隙可能跨越单个索引值、多个索引值，甚至可能为空。

间隙锁是性能和并发性之间权衡的一部分，并且被用于某些事务隔离级别，而不是其他级别。

使用唯一索引搜索唯一行的锁定语句不需要间隙锁。例如，如果id列有一个唯一的索引，下面的语句只对 id 值为100的行使用一个索引记录锁，其他会话是否在前面的间隙中插入行并不重要:
```
SELECT * FROM child WHERE id = 100;
```
如果 id 没有被索引或具有非唯一的索引，则语句将锁定前面的间隙。

这里还值得注意的是，冲突锁可能由不同的事务在一个间隙上持有。例如，事务A可以对间隙持有共享间隙锁(gap S-lock) ，而事务B对同一间隙持有排他间隙锁(gap X-lock)。 允许使用冲突的间隙锁的原因是，如果从索引中清除记录，则必须合并不同事务在记录上持有的间隙锁。

Innodb中的间隙锁是“完全禁止的”，这意味着它们的唯一目的是防止其他交易插入到间隙中。间隙锁可以共存。一个事务获取的间隙锁并不阻止另一个事务获取同一间隙的间隙锁。 共享间隙锁和独占间隙锁之间没有区别。它们之间没有冲突，而且它们执行相同的功能。

可以显式禁用间隙锁。如果您将事务隔离级别更改为<ins>READ COMMITTED</ins>，就会发生这种情况。 在这些情况下，搜索和索引扫描禁用间隙锁，并且仅用于外键约束检查和重复键检查。

使用 <ins>READ COMMITTED</ins> 隔离级别还有其他影响。 在 MySQL 评估 WHERE 条件之后，将释放不匹配行的记录锁。 对于 UPDATE 语句，InnoDB 执行“半一致(semi-consistent)”读操作，以便将最新提交的版本返回给 MySQL，这样 MySQL 就可以确定该行是否符合 UPDATE 的 WHERE 条件。

## 邻键锁(Next-Key Locks)
邻键锁是索引记录上的记录锁和索引记录前的间隙锁的组合。

InnoDB执行行级锁定的方式是，当它搜索或扫描表索引时，它会在遇到的索引记录上设置共享锁或排他锁。 因此，行级锁实际上是索引记录锁。索引记录上的邻键锁也会影响该索引记录之前的“间隙”。也就是说，邻键锁是索引记录锁加上索引记录前的间隙锁。 如果一个会话对索引中的记录R有一个共享或排他锁，那么另一个会话就不能按索引顺序在紧靠R的间隙中插入新的索引记录。

假设一个索引包含值10、11、13和20。该索引可能的邻键锁覆盖以下区间，其中圆括号表示排除了间隔端点，方括号表示包含端点:
```
(negative infinity, 10]
(10, 11]
(11, 13]
(13, 20]
(20, positive infinity)
```
对于最后一个间隔，邻键锁定索引中最大值以上的空隙，并锁定“上确界”伪记录的值高于索引中实际的任何值。上确界不是真正的索引记录，因此实际上，这个邻键锁只锁定最大索引值之后的空隙。

默认情况下，InnoDB 在 <ins>REPEATABLE READ</ins> 事务隔离级别运行。在这种情况下，InnoDB 使用邻键锁进行搜索和索引扫描，以防止幻象行。

邻键锁的事务数据类似于 <ins>SHOW ENGINE INNODB STATUS</ins> 和 INNODB 监视器输出中的以下内容:
```
RECORD LOCKS space id 58 page no 3 n bits 72 index `PRIMARY` of table `test`.`t` 
trx id 10080 lock_mode X
Record lock, heap no 1 PHYSICAL RECORD: n_fields 1; compact format; info bits 0
 0: len 8; hex 73757072656d756d; asc supremum;;

Record lock, heap no 2 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 8000000a; asc     ;;
 1: len 6; hex 00000000274f; asc     'O;;
2: len 7; hex b60000019d0110; asc        ;;
```

## 插入意向锁(Insert Intention Locks)
插入意图锁是由 INSERT 操作在插入行之前设置的一种间隙锁。 这个锁标志着插入的意向，以这样的方式，插入到同一索引间隙中的多个事务如果没有插入到间隙中的同一位置，则不需要彼此等待。假设有值为4和7的索引记录。 尝试插入值5和6的独立事务，每个事务在获得插入行的独占锁之前用插入意向锁锁锁定4和7之间的间隙，但不会阻塞彼此，因为行之间没有冲突。

下面的示例演示在获取所插入记录的独占锁之前使用插入意向锁的事务。这个例子涉及到两个客户端，A和B。

客户端A创建一个包含两个索引记录(90和102)的表，然后启动一个事务，该事务对 ID 大于100的索引记录放置排他锁。排他锁在记录102之前包含一个间隔锁:

```sql
mysql> CREATE TABLE child (id int(11) NOT NULL, PRIMARY KEY(id)) ENGINE=InnoDB;
mysql> INSERT INTO child (id) values (90),(102);

mysql> START TRANSACTION;
mysql> SELECT * FROM child WHERE id > 100 FOR UPDATE;
+-----+
| id  |
+-----+
| 102 |
+-----+
```
客户端B开始一个事务，将一个记录插入到间隙中。事务在等待获取排他锁时接受插入意图锁。
```sql
mysql> START TRANSACTION;
mysql> INSERT INTO child (id) VALUES (101);
```
插入意图锁的事务数据类似于 <ins>SHOW ENGINE INNODB STATUS</ins> 和 INNODB 监视器输出中的以下内容:
```
RECORD LOCKS space id 31 page no 3 n bits 72 index `PRIMARY` of table `test`.`child`
trx id 8731 lock_mode X locks gap before rec insert intention waiting
Record lock, heap no 3 PHYSICAL RECORD: n_fields 3; compact format; info bits 0
 0: len 4; hex 80000066; asc    f;;
 1: len 6; hex 000000002215; asc     " ;;
 2: len 7; hex 9000000172011c; asc     r  ;;...
```

## 自增锁(AUTO-INC Locks)
自增锁是一种特殊的表级锁，它由插入到带有 ```AUTO_INCREMENT``` 列的表中的事务获得。 在最简单的情况下，如果一个事务正在向表中插入值，那么任何其他事务都必须等待自己的插入操作，以便第一个事务插入的行接收连续的主键值。

该<ins>innodb_autoinc_lock_mode</ins> 配置选项控制用于自增锁的算法。它允许您选择如何在可预测自增值序列与插入操作的最大并发性之间进行权衡。

## 空间索引的谓词锁(Predicate Locks for Spatial Indexes)

Innodb 支持对包含空间列的列进行 SPATIAL 索引。

为了处理与 SPATIAL 索引有关的操作的锁定，邻键锁定不能很好地支持 <ins>REPEATABLE READ</ins> 或 <ins>SERIALIZABLE</ins> 事务隔离级别。 多维数据中没有绝对排序概念，因此不清楚哪个是邻键。

为了支持具有 SPATIAL 索引的表的隔离级别，InnoDB使用谓词锁。空间索引包含最小外接矩形值，因此 InnoDB 通过在用于查询的 MBR 值上设置谓词锁来强制对索引进行一致性读。 其他事务不能插入或修改与查询条件匹配的行。

