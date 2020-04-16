## 总结

1、FLUSH TABLES关闭所有打开的表，强制关闭所有正在使用的表，并刷新查询缓存和预准备语句缓存，不会刷新脏块 
2、FLUSH TABLES WITH READ LOCK关闭所有打开的表并使用全局读锁锁定所有数据库的所有表，不会刷新脏块 
3、如果一个会话中使用LOCK TABLES tbl_name lock_type语句对某表加了表锁，在该表锁未释放前，那么另外一个会话如果执行FLUSH TABLES语句会被阻塞，执行FLUSH TABLES WITH READ LOCK也会被堵塞 
4、如果一个会话正在执行DDL语句，那么另外一个会话如果执行FLUSH TABLES 语句会被阻塞 ，执行FLUSH TABLES WITH READ LOCK也会被堵塞 
5、如果一个会话正在执行DML大事务（DML语句正在执行，数据正在发生修改，而不是使用lock in share mode和for update语句来显式加锁），那么另外一个会话如果执行FLUSH TABLES语句会被阻塞，执行FLUSH TABLES WITH READ LOCK也会被堵塞 
6、FLUSH TABLES WITH READ LOCK语句不会阻塞日志表的写入，例如：查询日志，慢查询日志等 
7、mysqldump的--master-data、--lock-all-tables参数引发FLUSH TABLES和FLUSH TABLES WITH READ LOCK 
8、FLUSH TABLES tbl_name [, tbl_name] ... FOR EXPORT 会刷新脏块 
9、FLUSH TABLES WITH READ LOCK可以针对单个表进行锁定，比如只锁定table1则flush tables table1 with read lock;

## 详解FTWRL

FLUSH TABLES WITH READ LOCK简称(FTWRL)，该命令主要用于备份工具获取一致性备份(数据与binlog位点匹配)。**由于FTWRL总共需要持有两把全局的MDL锁，并且还需要关闭所有表对象，因此这个命令的杀伤性很大，执行命令时容易导致库hang住。**如果是主库，则业务无法正常访问；如果是备库，则会导致SQL线程卡住，主备延迟。本文将详细介绍FTWRL到底做了什么操作，每个操作的对库的影响，以及操作背后的原因。 

### FTWRL步骤

FTWRL主要包括3个步骤：

1.上全局读锁(lock_global_read_lock)
2.清理表缓存(close_cached_tables)
3.上全局COMMIT锁(make_global_read_lock_block_commit)

### FTWRL详解

上全局读锁会导致所有更新操作都会被堵塞；关闭表过程中，如果有大查询导致关闭表等待，那么所有访问这个表的查询和更新都需要等待；上全局COMMIT锁时，会堵塞活跃事务提交。

### 主备切换场景

在生产环境中，为了容灾一般mysql服务都由主备库组成，当主库出现问题时，可以切换到备库运行，保证服务的高可用。在这个过程中有一点很重要，避免双写。因为导致切换的场景有很多，可能是因为主库压力过大hang住了，也有可能是主库触发mysql bug重启了等。当我们将备库写开启时，如果老主库活着，一定要先将其设置为read_only状态。“set global read_only=1”这个命令实际上也和FTWRL类似，也需要上两把MDL，只是不需要清理表缓存而已。如果老主库上还有大的更新事务，将导致set global read_only hang住，设置失败。因此切换程序在设计时，要考虑这一点。

关键函数：fix_read_only

1.lock_global_read_lock(),避免新的更新事务，阻止更新操作
2.make_global_read_lock_block_commit，避免活跃的事务提交

### FTWRL和备份

Mysql的备份方式，主要包括两类，逻辑备份和物理备份，逻辑备份的典型代表是mysqldump，物理备份的典型代表是extrabackup。根据备份是否需要停止服务，可以将备份分为冷备和热备。冷备要求服务器关闭，这个在生产环境中基本不现实，而且也与FTWRL无关，这里主要讨论热备。

Mysql的架构支持插件式存储引擎，通常我们以是否支持事务划分，典型的代表就是myisam和innodb，这两个存储引擎分别是早期和现在mysql表的默认存储引擎。我们的讨论也主要围绕这两种引擎展开。对于innodb存储引擎而言，在使用mysqldump获取一致性备份时，我们经常会使用两个参数，**--single-transaction和--master-data，前者保证innodb表的数据一致性，后者保证获取与数据备份匹配的一致性位点，主要用于搭建复制。**现在使用mysql主备集群基本是标配，所以也是必需的。对于myisam，就需要通过--lock-all-tables参数和--master-data来达到同样的目的。我们在来回顾下FTWRL的3个步骤：

1. 上全局读锁
2. 清理表缓存
3. 上全局COMMIT锁

第一步的作用是堵塞更新，备份时，我们期望获取此时数据库的一致状态，不希望有更多的更新操作进来。对于innodb引擎而言，其自身的MVCC机制，可以保证读到老版本数据，因此第一步对它使多余的。第二步，清理表缓存，这个操作对于myisam有意义，关闭myisam表时，会强制要求表的缓存落盘，这对于物理备份myisam表是有意义的，因为物理备份是直接拷贝物理文件。对于innodb表，则无需这样，因为innodb有自己的redolog，只要记录当时LSN，然后备份LSN以后的redolog即可。第三步，主要是保证能获取一致性的binlog位点，这点对于myisam和innodb作用是一样的。

所以总的来说，FTWRL对于innodb引擎而言，最重要的是获取一致性位点，前面两个步骤是可有可无的，因此如果业务表全部是innodb表，这把大锁从原理上来讲是可以拆的，而且percona公司也确实做了这样的事情，具体大家可以参考[blog链接](https://www.percona.com/blog/2014/03/11/introducing-backup-locks-percona-server-2/)。此外，官方版本的5.5和5.6对于mysqldump做了一个优化，主要改动是，5.5备份一个表，锁一个表，备份下一个表时，再上锁一个表，已经备份完的表锁不释放，这样持续进行，直到备份完成才统一释放锁。5.6则是备份完一个表，就释放一个锁，实现主要是通过innodb的保存点机制。