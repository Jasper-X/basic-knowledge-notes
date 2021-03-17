# 简述 MySQL 三种日志的使用场景

## redo log

redo log 即为重做日志。为了最大程度的避免数据写入时，因为 IO 瓶颈造成的性能问题，MySQL 采用了这样一种缓存机制，先将数据写入内存中，再批量把内存中的数据统一刷回磁盘。为了避免将数据刷回磁盘过程中，因为掉电或系统故障带来的数据丢失问题，InnoDB 采用 redo log 来解决此问题。

`redo log` 包括两部分：一个是内存中的日志缓冲( redo log buffer )，另一个是磁盘上的日志文件( redo logfile)。
`mysql` 每执行一条 DML 语句，先将记录写入 redo log buffer，后续某个时间点再一次性将多个操作记录写到 redo log file。这种 **先写日志，再写磁盘**的技术就是 MySQL里经常说到的 WAL(Write-Ahead Logging) 技术。

在计算机操作系统中，用户空间(`user space`)下的缓冲区数据一般情况下是无法直接写入磁盘的，中间必须经过操作系统内核空间( kernel space )缓冲区( OS Buffer )。

#### 使用场景：

- crash-safe：将未刷到磁盘上的脏页变更应用到磁盘，保证了数据不丢。



## undo log

undo log 即为回滚日志。用于存储日志被修改前的值，从而保证如果修改出现异常，可以使用 undo log 日志来实现回滚操作。 undo log 和 redo log 记录物理日志不一样，它是逻辑日志，可以认为当 delete 一条记录时，undo log 中会记录一条对应的 insert 记录，反之亦然，当 update 一条记录时，它记录一条对应相反的 update 记录，当执行 rollback 时，就可以从 undo log 中的逻辑记录读取到相应的内容并进行回滚。undo log 默认存放在共享表空间中。



#### 使用场景：

- 事务回滚：undo log 记录了数据的逻辑变化，在事务回滚时，会根据 undo log 记录的旧版本数据进行回滚。
- MVCC：undo log 中记录了每个版本数据的 trx_id，根据 trx_id 找到对应版本的数据。





## bin log

bin log 即为二进制日志。用于记录数据库执行的写入性操作(不包括查询)信息，以二进制文件的形式保存在磁盘中。binlog 是 逻辑日志，并且由 Server 层进行记录，使用任何存储引擎的 mysql 数据库都会记录 binlog 日志。

binlog 是通过追加的方式进行写入的，可以通过max_binlog_size 参数设置每个 binlog 文件的大小，当文件大小达到给定值之后，会生成新的文件来保存日志。

#### 使用场景：

- 主从复制：在 `Master` 端开启 `binlog` ，然后将  `binlog` 发送到各个 `Slave` 端， `Slave` 端重放 `binlog` 从而达到主从数据一致。
- 数据恢复 ：通过使用 `mysqlbinlog` 工具来恢复数据。

#### 刷盘时机：

对于 `InnoDB `存储引擎而言，只有在事务提交时才会记录  `biglog` ，此时记录还在内存中。
`mysql`通过 `sync_binlog` 参数控制 `binlog` 的刷盘时机，取值范围是 `0-N`：

- 0：不去强制要求，由系统自行判断何时写入磁盘；
- 1：每次 commit 的时候都要将 binlog 写入磁盘；
- N：每N个事务，才会将 binlog 写入磁盘。

从上面可以看出，`sync_binlog`最安全的是设置是 `1` ，这也是`MySQL 5.7.7`之后版本的默认值。但是设置一个大一些的值可以提升数据库性能，因此实际情况下也可以将值适当调大，牺牲一定的一致性来获取更好的性能。

#### 日志格式：

`binlog` 日志有三种格式，分别为 `STATMENT` 、 `ROW` 和 `MIXED`。

> 在 MySQL 5.7.7 之前，默认的格式是 STATEMENT ， MySQL 5.7.7 之后，默认值是 ROW。日志格式通过 binlog-format 指定。

- STATMENT：基于SQL 语句的复制( statement-based replication, SBR )，每一条会修改数据的sql语句会记录到binlog 中 。
    - 优点：不需要记录每一行的变化，减少了 binlog 日志量，节约了 IO , 从而提高了性能；
    - 缺点：在某些情况下会导致主从数据不一致，比如执行sysdate() 、 sleep() 等 。
- ROW：基于行的复制(row-based replication, RBR )，不记录每条sql语句的上下文信息，仅需记录哪条数据被修改了 。
    - 优点：不会出现某些特定情况下的存储过程、或function、或trigger的调用和触发无法被正确复制的问题 ；
    - 缺点：会产生大量的日志，尤其是`alter table` 的时候会让日志暴涨
- MIXED：基于STATMENT 和 ROW 两种模式的混合复制(mixed-based replication, MBR )，一般的复制使用STATEMENT 模式保存 binlog ，对于 STATEMENT 模式无法复制的操作使用 ROW 模式保存 binlog