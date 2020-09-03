面试官：分别讲一下MySQL的几大文件，你懂的

我：我不懂，ok，好的。

- undoLog 也就是我们常说的**回滚日志文件** 主要用于事务中执行失败，进行回滚，以及MVCC中对于数据历史版本的查看。由**引擎层的InnoDB引擎实现**,是**逻辑日志**,记录数据修改被修改前的值,比如"把id='B' 修改为id = 'B2' ，那么undo日志就会用来存放id ='B'的记录”。当一条数据需要更新前,会先把修改前的记录存储在undolog中,如果这个修改出现异常,则会使用undo日志来实现回滚操作,保证事务的一致性。当事务提交之后，**undo log并不能立马被删除,而是会被放到待清理链表中,待判断没有事务用到该版本的信息时才可以清理相应undolog**。它保存了事务发生之前的数据的一个版本，用于回滚，**同时可以提供多版本并发控制下的读（MVCC）**，也即非锁定读。
- redoLog 是重做日志文件是**记录数据修改之后的值**，**用于持久化到磁盘中**。redo log包括两部分：**一是内存中的日志缓冲(redo log buffer)，该部分日志是易失性的**；二是**磁盘上的重做日志文件(redo log file)**，该部分日志是持久的。由引**擎层的InnoDB引擎实现**,是**物理日志**,记录的是物理数据页修改的信息,比如“某个数据页上内容发生了哪些改动”。当一条数据需要更新时,InnoDB会先将数据更新，然后记录redoLog 在内存中，然后找个时间将redoLog的操作执行到磁盘上的文件上。不管是否提交成功我都记录，你要是回滚了，那我连回滚的修改也记录。它确保了事务的持久性。
- binlog由**Mysql的Server层实现**,是**逻辑日志**,记录的是sql语句的原始逻辑，比如"把id='B' 修改为id = ‘B2’。binlog会写入指定大小的物理文件中,是追加写入的,当前文件写满则会创建新的文件写入。 产生:**事务提交的时候,一次性将事务中的sql语句,按照一定的格式记录到binlog中。用于复制和恢复在主从复制中，从库利用主库上的binlog进行重播(执行日志中记录的修改逻辑),实现主从同步。业务数据不一致或者错了，用binlog恢复**。 
- MVCC多版本并发控制是MySQL中基于**乐观锁理论实现隔离级别**的方式，用于**读已提交和可重复读**取隔离级别的实现。在MySQL中，会在表中每一条数据后面添加两个字段：**最近修改该行数据的事务ID**，**指向该行（undolog表中）回滚段的指针**。**Read View判断行的可见性，创建一个新事务时，copy一份当前系统中的活跃事务列表。意思是，当前不应该被本事务看到的其他事务id列表**。

**binlog和redolog的区别**：

1. redolog是在**InnoDB存储引擎层产生**，而**binlog是MySQL数据库的上层服务层产生**的。
2. 两种日志记录的内容形式不同。MySQL的**binlog是逻辑日志**，其记录是**对应的SQL语句**。而**innodb存储引擎层面的重做日志是物理日志**。
3. 两种日志与记录写入磁盘的时间点不同，**binlog日志只在事务提交完成后进行一次写入**。而**innodb存储引擎的重做日志在事务进行中不断地被写入，并日志不是随事务提交的顺序进行写入的**。
4. **binlog不是循环使用，在写满或者重启之后，会生成新的binlog文件，redolog是循环使用**。
5. **binlog可以作为恢复数据使用，主从复制搭建**，**redolog作为异常宕机或者介质故障后的数据恢复使用**。

**MVCC的缺点：**

MVCC在大多数情况下代替了行锁，实现了对读的非阻塞，读不加锁，读写不冲突。缺点是每行记录**都需要额外的存储空间，需要做更多的行维护和检查工作**。 要知道的，MVCC机制下，会在更新前建立undo log，根据各种策略读取时非阻塞就是MVCC，undo log中的行就是MVCC中的多版本。 而undo log这个关键的东西，**记载的内容是串行化的结果，记录了多个事务的过程，不属于多版本共存**。 这么一看，似乎mysql的mvcc也并没有所谓的多版本共存

**读写分离原理**：

主库（master）将变更写**binlog**日志，然后从库（slave）连接到主库之后，从库有一个**IO线程**，将主库的binlog日志**拷贝到自己本地**，写入一个中继日志中。接着从库中有一个SQL线程会从中继日志读取binlog，然后执行binlog日志中的内容，也就是在自己本地再次执行一遍SQL，这样就可以保证自己跟主库的数据是一样的。

这里有一个非常重要的一点，就是从库同步主库数据的过程是**串行化**的，也就是说**主库上并行**的操作，在从库上会串行执行。所以这就是一个非常重要的点了，由于从库从主库拷贝日志以及串行执行SQL的特点，在高并发场景下，从库的数据一定会比主库慢一些，是有延时的。所以经常出现，刚写入主库的数据可能是读不到的，要过几十毫秒，甚至几百毫秒才能读取到。

而且这里还有另外一个问题，就是如果主库突然宕机，然后恰好数据还没同步到从库，那么有些数据可能在从库上是没有的，有些数据可能就丢失了。

所以mysql实际上在这一块有两个机制，一个是**半同步复制**，用来解决主库数据丢失问题；一个是**并行复制**，用来解决主从同步延时问题。

所谓并行复制，指的是从库**开启多个线程，并行读取relay log中不同库的日志**，然后并行重放不同库的日志，这是库级别的并行。