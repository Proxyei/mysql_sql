# MySQL高级

# 1、MySQL架构分析

1. 连接层

2. 服务层

3. 引擎层

4. 存储层

   



# 2、**索引优化分析（重点）** 

### 1、SQL执行慢的原因

1. SQL语句问题
2. 索引失效
3. 表设计问题，太多的join查询

### 2、SQL解释执行的顺序

1. FROM
2. ON
3. JOIN
4. WHERE
5. GROUP BY
6. HAVING
7. SELECT
8. DESTINCT
9. ORDER BY
10. LIMIT

# 3、查询截取分析



# 4、MySQL锁机制

查看锁命令：

show open tables；

## 1、乐观锁

认为每次都不会被其他人修改数据，通过定义表字段version实现数据的CURD。

## 2、悲观锁

认为每次操作都会修改数据。**数据的查询默认不加锁，但是CUD会默认加排它锁 **  

### 1、共享锁（读锁）

加锁的会话可以进行

### 2、排它锁（写锁）

加锁的会话可以进行CURD，而其他会话无法进行CURD

## 3、行锁（偏向排它锁），支持事务

不是锁记录，而是锁索引。

一般指排它锁，锁定期间只允许其他会话读操作，不能写操作。行锁之前需要先加表锁。

分锁定主键索引与非主键索引。

操作非主键索引的时候，会先锁定非主键索引，再进行主键索引锁定，与表锁一样。

## 4、表锁（偏向表结构共享锁）

命令：

表加锁：lock table table1 read/write,table2 read/write;

表解锁：unlock tables ;

一般共享锁，锁定期间，DML操作不做限制，但是不可进行DDL操作。

## 5、myisam与innodb区别

1. ## myisam引擎进行写操作的时候，会进行表锁，导致性能低下，并发量不好。

2. myisam不存在死锁。

3. innodb支持事务，并发量好。

4. innodb使用行级锁。

5. myisam使用表锁。

## 6、复习一下事务

### 1、事务的四大特性：

原子性，一致性，隔离性，持久性

### 2、并发带来的事务问题：

1. 更新丢失（lost update）

   描述：多个事务同时修改一个行的时候，最后一个事务会覆盖其他的事务。

   解决：在一个事务提交之前，禁止其他事务访问。

   

2. 脏读（dirty reads）

   描述：事务B读取到了事务A已经**修改但是尚未提交**的数据，事务B还在此基础上修改了数据，当时如果A回滚事务，事务B读取的数据无效，无法做到一致性。

   

3. 不可重复读(non-repeatable reads)

   描述：事务B读取数据的时候，第二次读取原来的数据，发现原来的数据已经被修改了，简单说，也就是事务B读到了事务A**已修改已提交**的数据，不符合隔离性。

   

4. 幻读（phantom reads)

   描述：事务B读到了事务A**新增已提交**的数据

### 3、事务问题解决办法

#### 1、使用事务的隔离机制

| 级别（从高到低） |      | 脏读     | 不可重复读 | 幻读     |
| ---------------- | ---- | -------- | ---------- | -------- |
| 读尚未提交       |      | 会出现   | 会出现     | 会出现   |
| 读已提交         |      | 不会出现 | 会出现     | 会出现   |
| 重复读           |      | 不会出现 | 不会出现   | 会出现   |
| 可序列化         |      | 不会出现 | 不会出现   | 不会出现 |

级别越高，并发性能越低。

## 7，练习操作

## 1、myisam引擎（偏表锁）

### 1、emp表进行read表锁：

1. lock table read操作，当前会话可以进行读操作，但是无法进行写操作，而其他会话可以进行读操作，因为读写操作都会加锁，所以如果其他会话中有一个会话进行写操作的时候会堵塞，那么这一个会话进行写操作的时候，会等待锁，其他会话即使进行读操作也会堵塞，因为在等待前一个写操作释放锁（**共享锁**）。
2. 在unlock tables释放锁操作之前，无法对其他表进行任何操作。

### 2、dept表进行write表锁

1. lock table write操作的时候，当前会话可以进行读写操作，其他会话读写操作进行堵塞状态等待锁释放（**排它锁**）。
2. 在unlock tables释放锁操作之前，无法对其他表进行任何操作。

2、innodb引擎（偏行锁）

1. 两个session开始事务手动提交A，B：set autocommit=0，第三个会话C：set autocommit=1;
2.  A/B如果修改同一行，如果A尚未提交事务，则B修改会进入堵塞状态，如果读操作则不会。



# 5、主从复制（mysql8）

## 注意：

- master与slave的server-id、servier-uuid都不能相同，server_id、server_uuid分别在my.cnf，/${datadir}/auto.cnf下修改。
- master的binlog-ignore-db与slave的replication-ignore-db只能配一个（对应的binlog-do-db与replication-do-db有待考证）。

## 1、主要步骤

1. 打开master的my.cnf配置如下：

   server-id=1

   log-bin=mysql-bin

   binlog-do-db=database（如果要复制所有数据库，就不写）

   binlog-ignore-db=database

2. 设置master用于复制操作的账号

   create user 'master'@'%' identified by 'Master123'

   grant repliaction slave on \*.\* to  'master'@'%' 

   show master status（记下bin文件名字与position）

3. 打开slave配置my.cnf

   server-id=2

   log-bin=mysql-bin

4. 登录slave的mysql，设置

   change master to 

   master_host=master ip，

   master_user=master用于复制的账号，

   master_password=上面的密码,

   master-log-file=上面查出来的的master的bin文件,

   master-log-pos=上面查出来的position;

   接着show slave status/G;确定没有错误，确定slave的状态都是true。

5. 以上基本可以