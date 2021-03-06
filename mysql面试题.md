# mysql

## mysql的系统架构

mysql主要分为Server层和存储引擎层

server层主要包括连接器，缓存（mysql8.0之后移除），分析器，优化器，执行器。跨存储引擎层的功能都在这一层实现，比如存储过程（类似java中的方法），触发器，函数等。还有所有存储引擎层都可以用的binlog日志模块

存储引擎层负责数据的存储。 最常见的存储引擎有Innodb, myisyam。 其中innodb包含了一个readlog日志模块（因为有这个日志才支持事务）

![mysql系统架构](D:\课件\-\mysql系统架构.jpg)

## 一条查询是如何执行的

```sql
select * from tb_student  A where A.age='18' and A.name='张三';
```

1. 经过连接器校验该用户是否有权限， 没权限报错， 有权限的话查询出权限列表（比如是否允许写）
2. mysql8.0之前的版本会以sql语句（select * from tb_student  A where A.age='18' and A.name='张三'）为key，查询缓存。若存在则返回，否则继续
3. 分析器， 分析出要select，select的表为tb_student  ，要查询所有的列。。 继续判断语法是否有问题
4. 优化器，上边的sql有两种执行方案，优化器会选择最优的一种方案来执行
   1. 先查询学生表中姓名为“张三”的学生，然后判断是否年龄是18。  
   2. 先找出学生中年龄18岁的学生，然后再查询姓名为“张三”的学生。
5. 执行器，校验第一步查询出的权限列表，有权限调用存储引擎。否则报错

## 一条增删改语句时如何执行的

```sql
update tb_student A set A.age='19' where A.name='张三';
```

数据库肯定是不会存age字段的， 因为年龄会发生变化，所以会存出生日期

流程和查询是一样的。 只是需要涉及到事务

执行更新的时候肯定要记录日志啦，这就会引入日志模块了，mysql 自带的日志模块式binlog（归档日志），所有的存储引擎都可以使用，我们常用的InnoDB引擎还自带了一个日志模块redo log。

1. 先查询到张三这一条数据
2. 然后拿到查询的语句，把 age 改为19，然后调用引擎API接口，写入这一行数据，InnoDB引擎把数据保存在内存中，同时记录redo log，此时redo log进入prepare状态，然后告诉执行器，执行完成了，随时可以提交。
3. 执行器收到通知后记录binlog，然后调用引擎接口，提交redo log 为提交状态。
4. 更新完成。



为什么InnoDB存储引擎要有两个日志模块。这就是InnoDB存储引擎支持事务的原因。 而MyIsam不支持事务就是它没有自己的日志



为什么redo log 要引入prepare预提交状态？

反例：

- 先写readlog直接提交，然后写binlog。 假设readlog写完了机器挂了，binlog没有写入。 那么机器重启之后通过readlog恢复数据，但是binlog中并没有。就会造成数据丢失
- 先写binlog，然后写readlog。假设写完了binlog，机器重启了，由于没有readlog，本机是无法恢复这条数据的。但是binlog又有记录，数据不一致。

如果采用redo log 两阶段提交的方式就不一样了，写完binglog后，然后再提交redo log就会防止出现上述的问题，从而保证了数据的一致性。那么问题来了，有没有一个极端的情况呢？假设redo log 处于预提交状态，binglog也已经写完了，这个时候发生了异常重启会怎么样呢？ 这个就要依赖于mysql的处理机制了，mysql的处理过程如下：

- 判断redo log 是否完整，如果判断是完整的，就立即提交。
- 如果redo log 只是预提交但不是commit状态，这个时候就会去判断binlog是否完整，如果完整就提交 redo log, 不完整就回滚事务。



## 什么是索引

数据库索引，是数据库管理系统中一个排序的数据结构，以协助快速查询、更新数据库表中数据。就像我们以前用的新华字典的目录一样，能帮助我们快速查询到某一个字。



按照数据结构分： Hash索引， B+Tree索引

按照存储层面分： 聚簇索引， 非聚簇索引

按照索引类型分： 主键索引，唯一索引，普通索引，复合索引等

## 聚簇，非聚簇索引

聚簇索引：

表数据按照索引的顺序来存储的，叶子结点即存储了真实的数据行，不再有另外单独的数据页，在一张表上最多只能创建一个聚集索引。

非聚簇索引：

表数据存储顺序与索引顺序无关，叶子节点存储了数据所在的内存地址。



根据主键查询就是走聚簇索引，如果没有定义主键，那么他会选择一个唯一的非空索引代替。如果没有这样的索引，那么他会隐式的定义一个主键来作为聚簇索引。

## 索引的优缺点

### 优点

可以大大加快数据的检索速度，这也是创建索引的最主要的原因。

### 缺点

索引文件也会占用存储空间

索引会影响增删改的性能

## 什么样的字段适合创建索引

- 经常与其他表进行关联的列上需要创建索引
- 经常出现在where条件后的条件列需要创建索引
- 索引应该创建在区分粒度足够高的列上。（比如性别就不合适）
- 经常需要排序的列上需要创建索引
- 在一张表中不应该创建过多的索引。

```
User 表   id , name, time, sex, address, createTime

select * from User where id = 10;   聚簇索引
select * from User where name = '张三'; 非聚簇索引

select *  from User where id = 10 or age  = 18;   索引失效
```



## 什么情况会导致索引失效

- 如果查询条件中有or， 即使其中有索引列作为条件也不会走索引（PS: 只有or两边的条件都加了索引才能走索引）

- 对于多列索引，如果不是使用的第一部分则不会走索引（最左匹配原则）

  ```sql
  比如name age address 作为复合索引
  select * from user where name = &#39;张三&#39;     会走索引
  select * from user where name = &#39;张三&#39; and age = 18  会走索引
  select * from user where age = 18  不会走索引
  ```

- like查询以%开头不会走索引，%结束会走索引

  ```
  select * from user where name like '%三';   不会走
  select * from user where name like '三%';   走索引
  ```

  

- 如果mysql优化器认为全表扫描更快则不会走索引

- 如果索引列上使用函数则不会走索引

## 什么是数据库事务

事务是一组操作，要么都执行，要么都不执行。

```java
假如小明要给小红转账1000元，这个转账会涉及到两个关键操作就是：将小明的余额减少1000元，将小红的余额增加1000元。万一在这两个操作之间突然出现错误比如银行系统崩溃，导致小明余额减少而小红的余额没有增加，这样就不对了。事务就是保证这两个关键操作要么都成功，要么都要失败。
```

## 数据库的ACID

- 原子性 事务的原子性确保动作要么全部完成，要么完全不起作用；

- 一致性  执行事务前后，数据保持一致，多个事务对同一个数据读取的结果是相同的；

- 隔离性 并发访问数据库时，一个用户的事务不被其他事务所干扰，各并发事务之间数据库是独立的； 

  ``` 
  update user set name = '张三' where id = 10;
  ```

- 持久性  一个事务被提交之后。它对数据库中数据的改变是持久的，即使数据库发生故障也不应该对其有任何影响。

## 什么是脏读？幻读？不可重复读？

- 脏读(Drity Read)：某个事务已更新一份数据，另一个事务在此时读取了同一份数据，由于某些原因，前一个RollBack了操作，则后一个事务所读取的数据就会是不正确的。

- 不可重复读(Non-repeatable read):在一个事务的两次查询之中数据不一致，这可能是两次查询过程中间插入了一个事务更新的原有的数据。

- 幻读(Phantom Read):在一个事务的两次查询中数据笔数不一致，例如有一个事务查询了几列(Row)数据，而另一个事务却在此时插入了新的几列数据，先前的事务在接下来的查询中，就会发现有几列数据是它先前所没有的。

## 什么是事务的隔离级别？MySQL的默认隔离级别是什么？

数据库的隔离级别主要是解决以上的脏读，幻读，不可重复读

- **READ-UNCOMMITTED(读取未提交)：** 最低的隔离级别，允许读取尚未提交的数据变更，**可能会导致脏读、幻读或不可重复读**。

- **READ-COMMITTED(读取已提交)：** 允许读取并发事务已经提交的数据，**可以阻止脏读，但是幻读或不可重复读仍有可能发生**。

- **REPEATABLE-READ(可重复读)：** 对同一字段的多次读取结果都是一致的，除非数据是被本身事务自己所修改，**可以阻止脏读和不可重复读，但幻读仍有可能发生**。

- **SERIALIZABLE(可串行化)：** 最高的隔离级别，完全服从ACID的隔离级别。所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，**该级别可以防止脏读、不可重复读以及幻读**。

## **MyISAM和InnoDB存储引擎使用的锁：**

- MyISAM采用表级锁
- InnoDB支持行级锁和表级锁，默认为行级锁

## **行级锁，表级锁**

行级锁是Mysql中锁定粒度最细的一种锁，表示只针对当前操作的行进行加锁。行级锁能大大减少数据库操作的冲突。其加锁粒度最小，但加锁的开销也最大。



表级锁是MySQL中锁定粒度最大的一种锁，表示对当前操作的整张表加锁，它实现简单，资源消耗较少，被大部分MySQL引擎支持。最常使用的MYISAM与INNODB都支持表级锁定。

## 什么是死锁？

死锁是指两个或多个事务在同一资源上相互占用，并请求锁定对方的资源，从而导致恶性循环的现象。

## 数据库的乐观锁和悲观锁是什么？怎么实现的？

乐观锁和悲观锁是一种思想， java中也有。 java中的synchronized和lock就是悲观锁。 Atomic相关的类是乐观锁

- **悲观锁**：假定会发生并发冲突，屏蔽一切可能违反数据完整性的操作。在查询完数据的时候就把事务锁起来，直到提交事务。实现方式：使用数据库中的锁机制

  ```sql
  //核心SQL,主要靠for update
  select * from PP_PAY where reqNo = '100011482020082812215421N' for update
  ```

  

- **乐观锁**：假设不会发生并发冲突，只在提交操作时检查是否违反数据完整性。在修改数据的时候把事务锁起来，通过version的方式来进行锁定。实现方式：一般会使用版本号机制实现。

  ```sql
  update PP_PAY set status = '2', version = 0 + 1 where  = '100011482020082812215421N' and version= 0
  ```

  

## sql优化

- 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引
- SELECT子句中避免使用*号，尽量全部大写SQL
- 应尽量避免在 where 子句中对字段进行 is null 值判断，否则将导致引擎放弃使用索引而进行全表扫描，使用 IS NOT NULL
- where 子句中使用 or 来连接条件，也会导致引擎放弃使用索引而进行全表扫描
- in 和 not in 也要慎用，否则会导致全表扫描

## 你怎么知道SQL语句性能是高还是低

- 使用explain查看sql语句的执行计划

  ```sql
  explain select * from user where userId = 16
  ```

## 主从复制

将主数据库中的DDL和DML操作通过二进制日志（BINLOG）传输到从数据库上，然后将这些日志重新执行（重做）；从而使得从数据库的数据与主数据库保持一致。

优点：

- 主数据库出现问题，可以切换到从数据库。
- 可以进行读写分离（主库负责写， 从库负责读）

## 读写分离

读写分离是依赖于主从复制。master负责写， slave负责读

## mysql的分库分表

- 垂直分库， 按照不同的业务拆分为不同的库，比如订单相关的表存到订单数据库。支付相关的表存到支付数据库
- 水平分库， 比如支付表数据量过大， 查询过慢时可以拆分为多个支付表，数据结构是一模一样的分布到不同的数据库进行存储。

## 历史归档	

5000W

500w   只保存15天的数据。。。。    大数据平台