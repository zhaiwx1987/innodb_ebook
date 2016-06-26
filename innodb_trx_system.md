##前言

在前面几期关于InnoDB Redo和Undo实现的铺垫后，本节我们从上层的角度来阐述InnoDB的事务子系统是如何实现的，涉及的内容包括：InnoDB的事务相关模块，如何实现MVCC及ACID，如何进行事务的并发控制，事务系统如何进行管理等相关知识。本文的目的是让读者对事务系统有一个较全面的理解。

由于不同版本对事务系统都有改变，本文的所有分析基于当前GA的最新版本MySQL5.7.9，但也会在阐述的过程中，顺带描述之前版本的一些内容。本文也会介绍5.7版本对事务系统的一些优化点。

另外尽管InnoDB锁系统和事务有着非常密切的联系，但鉴于本文主要介绍事务模块，并且计划中的篇幅已经足够长。而锁系统又是一个非常复杂的模块，将在后面的月报中单独开一篇文章来讲述。

在阅读本文之前，强烈建议先阅读下之前两节的内容，因为事务系统和这些模块有着非常紧密的联系：

[MySQL · 引擎特性 · InnoDB undo log 漫游](http://mysql.taobao.org/monthly/2015/04/01/)
[MySQL · 引擎特性 · InnoDB redo log漫游](http://mysql.taobao.org/monthly/2015/05/01/)
[MySQL · 引擎特性 · InnoDB 崩溃恢复过程](http://mysql.taobao.org/monthly/2015/06/01/)

##开启事务

InnoDB提供了多种方式来开启一个事务，最简单的就是以一条BEGIN语句开始，也可以以START TRANSACTION开启事务，你还可以选择开启一个只读事务还是读写事务。所有显式开启事务的行为都会隐式的将上一条事务提交掉。

所有显示开启事务的入口函数均为trans_begin，如下列出了几种常用的事务开启方式。

### BEGIN 
当以BEGIN开启一个事务时，首先会去检查是否有活跃的事务还未提交，如果没有提交，则调用`ha_commit_trans`提交之前的事务。并释放之前事务持有的MDL锁。
执行BEGIN命令并不会真的去引擎层开启一个事务，仅仅是为当前线程设定标记，表示为显式开启的事务。

和BEGIN等效的命令还有“BEGIN WORK”及“START TRANSACTION”

###START TRANSACTION READ ONLY
使用该选项开启一个只读事务，当以这种形式开启事务时，会为当前线程的`thd->tx_read_only`设置为true。当Server层接受到任何数据更改的SQL时，都会直接拒绝请求，返回错误码`ER_CANT_EXECUTE_IN_READ_ONLY_TRANSACTION`，不会进入引擎层。

这个选项可以强约束一个事务为只读的，而只读事务在引擎层可以走优化过的逻辑，相比读写事务的开销更小，例如不用分配事务id，不用分配回滚段，不用维护到全局事务链表中。

该事务开启的方式从5.6版本开始引入。我们知道，在MySQL5.6版本中引入的一个对事务模块的重要优化：将全局事务链表拆成了两个链表：一个用于维护只读事务，一个用于维护读写事务。这样我们在构建一个一致性视图时，只需要遍历读写事务链表即可。但是在5.6版本中，InnoDB并不具备事务从只读模式自动转换成读写事务的能力，因此需要用户显式的使用以下两种方式来开启只读事务：
* 执行START TRANSACTION READ ONLY
* 或者将变量tx_read_only设置为true

5.7版本引入了模式自动转换的功能，但该语法依然保留了。

另外一个有趣的点是，在5.7版本中，你可以通过设置`session_track_transaction_info`变量来跟踪事务的状态，这货主要用于官方的分布式套件(例如fabric)，例如在一个负载均衡系统中，你需要知道哪些statement开启或处于一个事务中，哪些statement允许连接分配器调度到另外一个connection。。只读事务是一种特殊的事务状态，因此也需要记录到线程的`Transaction_state_tracker`中。 

关于Session tracker，可以参阅官方[WL#6631](http://dev.mysql.com/worklog/task/?id=6631)

###START TRANSACTION READ WRITE
和上述相反，该SQL用于开启读写事务。这也是默认的事务模式。但有一点不同的是，如果当前实例的read_only打开了且当前连接不是超级账户，则显示开启读写事务会报错。

同样的事务状态`TX_READ_WRITE`也要加入到Session Tracker中。 另外包括上述几种显式开启的事务，其标记`TX_EXPLICIT`也加入到session tracker中。

读写事务并不意味着一定在引擎层就被认定为读写事务了，5.7版本InnoDB里总是默认一个事务开启时的状态为只读的。举个简单的例子，如果你事务的第一条SQL是只读查询，那么在InnoDB层，它的事务状态就是只读的，如果第二条SQL是更新操作，就将事务转换成读写模式。

###START TRANSACTION WITH CONSISTENT SNAPSHOT

和上面几种方式不同的是，在开启事务时还会顺便创建一个视图（Read View），在InnoDB中，视图用于描述一个事务的可见性范围，也是多版本特性的重要组成部分。

这里会进入InnoDB层，调用函数`innobase_start_trx_and_assign_read_view`，注意只有你的隔离级别设置成REPEATABLE READ（可重复读）时，才会显式开启一个Read View，否则会抛出一个warning。

使用这种方式开启事务时，事务状态已经被设置成ACTIVE的。

状态变量`TX_WITH_SNAPSHOT`会加入到Session Tracker中。

###AUTOCOMMIT = 0
当autocommit设置成0时，就无需显式开启事务，如果你执行多条SQL但不显式的调用COMMIT（或者执行会引起隐式提交的SQL）进行提交，事务将一直存在。通常我们不建议将该变量设置成0，因为很容易由于程序逻辑或使用习惯造成事务长时间不提交。而事务长时间不提交，在MySQL里简直就是噩梦，各种诡异的问题都会纷纷出现。一种典型的场景就是，你开启了一条查询，但由于未提交，导致后续对该表的DDL堵塞住，进而导致随后的所有SQL全部堵塞，简直就是灾难性的后果。

另外一种情况是，如果你长时间不提交一个已经构建Read View的事务，purge线程就无法清理一些已经提交的事务锁产生的undo日志，进而导致undo空间膨胀。具体的表现为ibdata文件疯狂膨胀。我们曾在线上观察到好几百G的Ibdata文件。

**TIPS**：所幸的是从5.7版本开始提供了可以在线truncate undo log的功能，前提是开启了独立的undo表空间，并保留了足够的undo回滚段配置（默认128个），至少需要35个回滚段。其truncate原理也比较简单：当purge线程发现一个undo文件超过某个定义的阀值时，如果没有活跃事务引用这个undo文件，就将其设置成不可分配，并直接物理truncate文件。


##提交事务

事务的提交分为两种方式，一种是隐式提交，一种是显式提交。

当你显式开启一个新的事务，或者执行一条非临时表的DDL语句时，就会隐式的将上一个事务提交掉。另外一种就是显式的执行“COMMIT” 语句来提交事务

然而， 在不同的场景下，MySQL在提交时进行的动作并不相同，这主要是因为MySQL是一种服务器层-引擎层的架构，并存在两套日志系统：Binary log及引擎事务日志。MySQL支持两种XA事务方式：隐式XA和显式XA；当然如果关闭binlog，并且仅使用一种事务引擎，就没有XA可言了。

关于隐式XA的控制对象，在实例启动时决定使用何种XA模式，如下代码段：

```
  if (total_ha_2pc > 1 || (1 == total_ha_2pc && opt_bin_log))
  {
    if (opt_bin_log)
      tc_log= &mysql_bin_log;
    else
      tc_log= &tc_log_mmap;
  }
```

* 若打开binlog，且使用了事务引擎，则XA控制对象为mysql_bin_log
* 若关闭了binlog，且存在不止一种事务引擎时，则XA控制对象为tc_log_mmap
* 其他情况，使用tc_log_dummy，这种场景下就没有什么XA可言了，无需任何协调者来进行XA。

这三者是`TC_LOG`的子类，关系如下图所示：
![1](http://img1.tbcdn.cn/L1/461/1/76ab5026558d31d9d761d58b5e08f309278f6c8b.png)

具体的，包含以下几种类型的XA（不对数据产生变更的只读事务无需走XA）

###Binlog/Engine XA 

当开启binlog时, MySQL默认使用该隐式XA模式。 在5.7版本中，事务的提交流程包括：

**- Binlog Prepare：**
设置`thd->durability_property= HA_IGNORE_DURABILITY`, 表示在innodb prepare时，不刷redo log

**- InnoDB Prepare** （入口函数`innobase_xa_prepare --> trx_prepare`）：
更新InnoDB的undo回滚段，将其设置为Prepare状态（`TRX_UNDO_PREPARED`） 

**- 进入组提交**(`ordered_commit`)

Flush Stage： 此时形成的一组队列，有leader依次为别的线程写binlog文件
在准备写binlog前，会调用ha_flush_logs接口，将存储的日志写到最新的LSN，然后再写binlog到文件。这样做的目的是为了提升组提交的效率，具体参阅之我写的[一篇月报]( http://mysql.taobao.org/index.php?title=MySQL%E5%86%85%E6%A0%B8%E6%9C%88%E6%8A%A5_2015.01#MySQL_.C2.B7_.E6.80.A7.E8.83.BD.E4.BC.98.E5.8C.96.C2.B7_Group_Commit.E4.BC.98.E5.8C.96)

Sync Stage： 如果sync_binlog计数超过配置值，则进行一次文件fsync，注意，参数sync_binlog的含义不是指的这么多个事务之后做一次fsync，而是这么多“组”事务队列后做一次fsync。

Semisync Stage (RDS MySQL only)：如果我们在事务commit之前等待备库ACK（设置成AFTER_SYNC模式），用户线程会释放上一个stage的锁，并等待ACk。这意味着在等待ACK的过程中，我们并不堵塞上一个stage的binlog写入，可以增加一定的吞吐量。

Commit Stage：队列中的事务依次进行innodb commit，将undo头的状态修改为`TRX_UNDO_CACHED`/`TRX_UNDO_TO_FREE`/`TRX_UNDO_TO_PURGE`任意一种 (undo相关知识参阅之前的月报)；并释放事务锁，清理读写事务链表、readview等一系列操作。每个事务在commit阶段也会去更新事务页的binlog位点。

**TIPS**：如果你关闭了binlog_order_commits选项，那么事务就各自进行提交，这种情况下不能保证innodb commit顺序和binlog写入顺序一致，这不会影响到数据一致性，在高并发场景下还能提升一定的吞吐量。但可能影响到物理备份的数据一致性，例如使用xtrabackup（而不是基于其上的innobackup脚本）依赖于事务页上记录的binlog位点，如果位点发生乱序，就会导致备份的数据不一致。


###Engine/Engine XA

当binlog关闭时，如果事务跨引擎了，就可以在事务引擎间进行XA了，典型的例如InnoDB和Tokudb（在RDS MySQL里已同时支持这两种事务引擎）。当支持超过1种事务引擎时，并且binlog关闭了，就走TC LOG MMAP逻辑。对应的XA控制对象为tc_log_mmap。

由于需要持久化事务信息以用于重启恢复，因此在该场景下，tc_log_mmap模块会创建一个文件，名为tc.log，文件初始化大小为24KB，使用mmap的方式映射到内存中。

tc.log 以PAGE来进行划分，每个PAGE大小为8K，至少需要3个PAGE，初始化的文件大小也为3个PAGE（`TC_LOG_MIN_SIZE`），每个Page对应的结构体对象为st_page，因此需要根据page数，完成文件对应的内存控制对象的初始化。初始化第一个page的header，写入magic number以及当前的2PC引擎数（也就是`total_ha_2pc`）

下图描述了tc.log的文件结构：
![2](http://img3.tbcdn.cn/L1/461/1/1dac55fabdb1b408913388d31b6fcd498343320b.png)

在事务执行的过程中，例如遇到第一条数据变更SQL时，会注册一个唯一标识的XID（实际上通过当前查询的query_id来唯一标识），之后直到事务提交，这个xid都不会改变。事务引擎本身在使用undo时，必须加上这个XID标识。

在进行事务Prepare阶段，若事务涉及到多个引擎，先在各自引擎里做事务Prepare。

然后进入commit阶段，这时候会将xid记录到tc.log中（如上图所示），这类涉及到相对复杂的page选择流程，这里不展开描述，具体的参阅函数`TC_LOG_MMAP::commit`

在完成记录到tc.log后，就到引擎层各自提交事务。这样即使在引擎提交时失败，我们也可以在crash recovery时，通过读取tc.log记录的xid，指导引擎层相符合xid的事务进行提交。

###Engine Commit

当关闭binlog时，且事务只使用了一个事务引擎时，就无需进行XA了，相应的事务commit的流程也有所不同。

首先事务无需进入Prepare状态，因为对单引擎事务做XA没有任何意义。

其次，因为没有Prepare状态的保护，事务在commit时需要对事务日志进行持久化。这样才能保证所有成功返回的事务变更， 能够在崩溃恢复时全部完成。

###显式XA

MySQL支持显式的开启一个带命名的XA事务，例如：

```
XA BEGIN "xidname"
     do something.....
XA END 'xidname'
XA PREPARE 'xidname'   // 当完成这一步时，如果崩溃恢复，是可以在启动后通过XA RECOVER获得事务信息，并进行显式提交。
XA COMMIT 'xidname'  // 完全提交事务 
```

一个有趣的问题是，在5.7之前的版本中，如果执行XA的过程中，在完成XA PREPARE后，如果kill掉session，事务就丢失了，而不是像崩溃恢复那样，可以直接恢复出来。这主要是因为MySQL对Kill session的行为处理是直接回滚事务。

为了解决这个问题，MySQL5.7版本做了不小的改动，将XA的两阶段都记录到了binlog中。 这样状态是持久化了的，一次干净的shutdown后，可以通过扫描binlog恢复出XA事务的状态，对于kill session导致的XA事务丢失，逻辑则比较简单：内存中使用一个transaction_cache维护了所有的XA事务，在断开连接调用THD::cleanup时不做回滚，仅设置事务标记即可。

具体的参阅我之前写的[这篇月报](http://mysql.taobao.org/monthly/2015/04/05/)

###事务回滚

当由于各种原因（例如死锁，或者显式ROLLBACK）需要将事务回滚时，会调用handler接口`ha_rollback_low`，进而调用InnoDB函数`trx_rollback_for_mysql`来回滚事务。 回滚的方式是提取undo日志，做逆向操作。

由于InnoDB的undo是单独写在表空间中的，本质上和普通的数据页是一样的。如果在事务回滚时，undo页已经被从内存淘汰，回滚操作（特别是大事务变更回滚）就可能伴随大量的磁盘IO。因此InnoDB的回滚效率非常低。有的数据库管理系统，例如PostgreSQL，通过在数据页上冗余数据产生版本链的方式来实现多版本，因此回滚起来非常方便，只需要设置标记即可，但额外带来的问题就是无效数据清理开销。

###SavePoint管理

在事务执行的过程中，你可以通过设置SAVEPOINT的方式来管理事务的执行过程。

在介绍Savepoint之前，需要先介绍下trx_t::undo_no。在事务每次成功写入一次undo后，这个计数都会递增一次（参阅函数trx_undo_report_row_operation）。事务的undo_no也会记录到undo page中进行持久化。因此在undo链表上的undo_no总是有序递增的。

总的来说，主要有以下几种操作类型。

**设置SAVEPOINT**

语法： SAVEPOINT sp_name

入口函数：`trans_savepoint`

在事务中设置一个SAVEPOINT，你可以随意命名一个名字，在事务中设置的所有savepoint实际上维护了两份链表，一份挂在THD变量上（thd->get_transaction()->m_savepoints），包含了基本的savepoint信息及到引擎层的映射，另一份在引擎层的事务对象上（维持链表trx_t::trx_savepoints中）。

如下图所示：
![3](http://img3.tbcdn.cn/L1/461/1/462aeb96e933cd472a65876fc266611979214688.png)

总共分为以下几步：
0. 在增加新的SAVEPOINT时，总是先判断下是否同名的SAVEPOINT已经存在，如果存在，就用后者替换前者。
1. Server层维护的savepoint信息记录了命名信息及MDL锁的savepoint点。其中MDL锁的savepoint，可以实现回滚操作时释放该savepoint之后再获得的MDL锁。
2. 在当前线程的Binlog cache中写入设置Savepoint的SQL, 并保存binlog cache中的位点 （`binlog_savepoint_set`）。
3. 引擎层的savepoint中记录了最近一次的`trx_t::undo_no`及SAVEPOINT名字。通过这些信息可以准确的定位在设置SAVEPOINT点时Undo位点。（参阅引擎层入口函数：`trx_savepoint_for_mysql`）

**回滚SAVEPOINT**

语法： ROLLBACK TO [ SAVEPOINT ] sp_name
入口函数：`trans_rollback_to_savepoint`

检查点的回滚主要包括：
0. 如果事务是一个XA事务，且已经处于XA PREPARE状态时是不允许回滚到某个SAVEPOINT的。
1. 如果涉及非事务引擎，在binlog中写入回滚SQL，否则直接将binlog cache truncate到之前设置sp时保存的位点。 （binlog_savepoint_rollback）
2. 在引擎层进行回滚（trx_rollback_to_savepoint_for_mysql）
根据之前记录的undo_no，可以逆向操作当前事务占用的undo slot上的undo记录来进行回滚。
3. 判断是否允许回滚MDL锁：
* binlog关闭的情况下，总是允许回滚MDL锁
* 或者由引擎来确认（ha_rollback_to_savepoint_can_release_mdl），同时满足：
     * InnoDB：如果当前事务不持有任何事务锁（表级或者行级），则认为可以回滚MDL锁。
     * Binlog：如果没有更改非事务引擎，则可以释放MDL锁。
如果允许回滚MDL，则通过之前记录的`st_savepoint::mdl_savepoint`进行回滚


**释放SAVEPOINT**

语法为： RELEASE SAVEPOINT sp_name
入口函数：`trans_rollback_to_savepoint`

顾名思义，就是删除一个SAVEPOINT，操作也很简单，直接根据命名从server层和innodb层的清理掉，并释放对应的内存。

**隐式SAVEPOINT**

在InnoDB中，还有一种隐式的savepoint，通过变量`trx_t::last_sql_stat_start`来维护。

初始状态下`trx_t::last_sql_stat_start`的值为0，当执行完一条SQL时，会调用函数`trx_mark_sql_stat_end`将当前的`trx_t::undo_no`保存到`trx_t::last_sql_stat_start`中。

如果SQL执行失败，就可以据此进行statement级别的回滚操作（`trx_rollback_last_sql_stat_for_mysql`）。

无论是显式SAVEPOINT还是隐式SAVEPOINT，都是通过undo_no来指示回滚到哪个事务状态。

**两个有趣的bug**

[bug#79493](http://bugs.mysql.com/bug.php?id=79493)

在一个只读事务中，如果设置了SAVEPOINT，任意执行一次`ROLLBACK TO SAVEPOINT`都会将事务从只读模式改变成读写模式。这主要是因为在活跃事务中执行ROLLBACK 操作会强制转换READ-WRITE模式。实际上这是没必要的，因为并没有造成任何的数据变更。

[bug#79596](http://bugs.mysql.com/bug.php?id=79596)

这个bug可以认为是一个相当严重的bug：在一个活跃的做过数据变更操作的事务中，任意执行一次ROLLBACK TO SAVEPOINT（即使SAVEPOINT不存在），然后kill掉客户端，会发现事务却提交了，并且没有写到binlog中。这会导致主备的数据不一致。

重现步骤如下：

```
mysql> create table test (value int) engine=innodb;
Query OK, 0 rows affected (3.88 sec)

mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into test set value = 1;
Query OK, 1 row affected (4.43 sec)

mysql> rollback to savepoint tx_0;
ERROR 1305 (42000): SAVEPOINT tx_0 does not exist
mysql> Killed
```

最后一步直接对session的进程kill -9时会导致事务commit。这主要是因为如果直接kill客户端，服务器端在清理线程资源，进行事务回滚时，相关的变量并没有被重设，thd的command类型还是`SQLCOM_ROLLBACK_TO_SAVEPOINT`，在函数`MYSQL_BIN_LOG::rollback`函数中将不会调用`ha_rollback_low`的引擎层回滚逻辑。 原因是回滚到某个savepoint有特殊的处理流程，

如果是通过ctrl+c的方式关闭client段，实际上会发送一个类型为`COM_QUIT`的command，它会将`thd->lex->sql_command`设置为`SQLCOM_END`，这时候会走正常的回滚逻辑。


##事务执行管理

在事务执行的过程中，需要多个模块来辅助事务的正常执行：
* Server层的MDL锁模块，维持了一个事务过程中所有涉及到的表级锁对象。通过MDL锁，可以堵塞DDL，避免DDL和DML记录binlog乱序；
* InnoDB的trx_sys子系统，维持了所有的事务状态，包括活跃事务，非活跃事务对象，读写事务链表，负责分配事务id，回滚段，Readview等信息，是事务系统的总控模块；
* InnoDB的lock_sys子系统，维护事务锁信息，用于对修改数据操作做并发控制，保证了在一个事务中被修改的记录，不可以被另外一个事务修改；
* InnoDB的log_sys子系统，负责事务redo日志管理模块，
* InnoDB的purge_sys子系统，则主要用于在事务提交后，进行垃圾回收，以及数据页的无效数据清理。 

总的来说，事务管理模块的架构图，如下图所示：
![4](http://img3.tbcdn.cn/L1/461/1/0013974890f4841c42684af0b28a18f0ec09d6c0.png)

下面就几个事务模块的关键点展开描述。

###事务ID

在InnoDB一直维持了一个不断递增的整数，存储在`trx_sys->max_trx_id`中；每次开启一个新的读写事务时，都将该id分配给事务，同时递增全局计数。事务id可以看做一个事务的唯一标识。

在MySQL5.6及之前的版本中，总是为事务分配ID。但实际上这是没有必要的，毕竟只有做过数据更改的读写事务，我们才需要去根据事务id判断可见性。因此在MySQL5.7版本中，只有读写事务才会分配事务id，只读事务的id默认为0。

那么问题来了，怎么去区分不同的只读事务呢 ？ 这里在需要输出事务id时（例如执行`SHOW ENGINE INNODB STATUS` 或者查询INFORMATION_SCHEMA.INNODB_TRX表），使用只读事务对象的指针或上一个常量来标识其唯一性，具体的计算方式见函数`trx_get_id_for_print`。 所以如果你show出来的事务id看起来数字特别庞大， 千万不要惊讶。

对于全局最大事务id，每做256次赋值(`TRX_SYS_TRX_ID_WRITE_MARGIN`)就持久化一次到ibdata的事务页(`TRX_SYS_PAGE_NO`)中。

已分配的事务Id会加入到全局读写事务Id集合中（`trx_sys->rw_trx_ids`），事务Id和事务对象的map加入到`trx_sys->rw_trx_set`中，这是个有序的集合(`std::set`)，可以用于通过trx id快速定位到对应的事务对象。

事务分配得到的ID并不是立刻就被使用了，而是在做了数据修改时，需要创建或重用一个undo slot时，会将当前事务的id写入到undo page头，状态为`TRX_UNDO_ACTIVE`。这也是崩溃恢复时，InnoDB判断是否有未完成事务的重要依据。

在执行数据更改的过程中，如果我们更新的是聚集索引记录，事务ID + 回滚段指针会被写到聚集索引记录中，其他会话可以据此来判断可见性以及是否要回溯undo链。
对于普通的二级索引页更新,  则采用回溯聚集索引页的方式来判断可见性（如果需要的话）。关于MVCC，后文会有单独描述。

###事务链表和集合

事务子系统维护了三个不同的链表，用来管理事务对象 

**trx_sys->mysql_trx_list**
包含了所有用户线程的事务对象，即使是未开启的事务对象，只要还没被回收到trx_pool中，都被放在该链表上。当session断开时，事务对象从链表上摘取，并被回收到trx_pool中，以待重用。

**trx_sys->rw_trx_list**
读写事务链表，当开启一个读写事务，或者事务模式转换成读写模式时，会将当前事务加入到读写事务链表中，链表上的事务是按照`trx_t::id`有序的；在事务提交阶段将其从读写事务链表上移除。

**trx_sys->serialisation_list**
序列化事务链表，在事务提交阶段，需要先将事务的undo状态设置为完成，在这之前，获得一个全局序列号`trx->no`，从`trx_sys->max_trx_id`中分配，并将当前事务加入到该链表中。随后更新undo等一系列操作后，因此进入提交阶段的事务并不是trx->id有序的，而是根据trx->no排序。当完成undo更新等操作后，再将事务对象同时从serialisation_list和rw_trx_list上移除。

这里需要说明下`trx_t::no`，这是个不太好理清的概念，从代码逻辑来看，在创建readview时，会用到序列化链表，链表的第一个元素具有最小的`trx_t::no`，会赋值给`ReadView::m_low_limit_no`。purge线程据此创建的readview，只有小于该值的undo，才可以被purge掉。

总的来说，`mysql_trx_list`包含了`rw_trx_list`上的事务对象，`rw_trx_list`包含了`serialisation_list`上的事务对象。

事务ID集合有两个：
**trx_sys->rw_trx_ids**
记录了当前活跃的读写事务ID集合，主要用于构建ReadView时快速拷贝一个快照

**trx_sys->rw_trx_set**
这是<trx_id, trx_t>的映射集合，根据trx_id排序，用于通过trx_id快速获得对应的事务对象。一个主要的用途就是用于隐式锁转换，需要为记录中的事务id所对应的事务对象创建记录锁，通过该集合可以快速获得事务对象

###事务回滚段

对于普通的读写事务，总是为其指定一个回滚段（默认128个回滚段）。而对于只读事务，如果使用到了InnoDB临时表，则为其分配(1~32)号回滚段。（回滚段指定参阅函数`trx_assign_rseg_low`）

当为事务指定了回滚段后，后续在事务需要写undo页时，就从该回滚段上分别分配两个slot，一个用于`update_undo`，一个用于`insert_undo`。分别处理的原因是事务提交后，update_undo需要purge线程来进行回收，而insert_undo则可以直接被重利用。

关于undo相关知识可以参阅之前的[月报](http://mysql.taobao.org/monthly/2015/04/01/)

###事务引用计数

在介绍事务引用计数之前，我们首先要了解下什么是隐式锁。所谓隐式锁，其实并不是一个真正的事务锁对象，可以理解为一个标记：对于聚集索引页的更新，记录本身天然带事务id，对于二级索引页，则在page上记录最近一次更新的最大事务id，通过回表的方式判断可见性。

由于事务锁涉及到全局资源，创建锁的开销高昂，InnoDB对于新插入的记录，在没有冲突的情况下是不创建记录锁的。举个例子，Session 1插入一条记录，并保持未提交状态。另外一个session想更新这条记录，从数据页上读取到这条记录后，发现对应的事务id还处于活跃状态，根据当前的并发规则，这个更新需要被阻塞住。因此第二个session需要为session 1创建一条记录锁，然后将自己放入等待队列中。

在MySQL5.7版本之前，隐式锁转换的逻辑为（函数`lock_rec_convert_impl_to_expl`）
1. 首先判断记录对应的事务ID是否还处于活跃状态
     聚集索引： `lock_clust_rec_some_has_impl`
     二级索引： `lock_sec_rec_some_has_impl`
 如果不活跃，说明事务已提交，我们可以对这条记录做任何更改操作。直接返回
否则返回获取的trx_id
2. 持有lock_sys->mutex；
3. 持有trx_sys->mutex ，并获取当前记录中的事务id对应的内存事务对象trx_t
4. 为该事务创建一个锁对象，并加入到锁队列中。
5. 释放lock_sys->mutex

上述流程中长时间持有`lock_sys->mutex`，目的是防止在为其转换隐式锁为显式锁时事务被提交掉。尤其是在第三步，同时持有两把大锁去查找事务对象。在5.6官方版本中，这种查找操作还需要遍历链表，开销巨大，推高了临界资源的竞争。 

因此在5.7中引入事务计数`trx_t::n_ref`来辅助判断，在隐式锁转换时，通过读写事务集合(`rw_trx_set`)快速获得事务对象，同时对`trx_t::n_def`递增。这个过程无需加`lock_sys->mutex`锁。随后再持有Lock_sys->mutex去创建显式锁。在完成创建后，递减trx_t::n_ref。

为了防止为一个已提交的事务创建显式锁；在事务提交阶段也做了处理：在事务释放事务锁之前，如果引用计数非0，则表示有人正在做隐式锁转换，这里需要等待其完成。(参考函数`lock_trx_release_locks`)。

实际上上述修改是在官方优化读写事务链表之前完成的。由于在5.7里已经使用一个有序的集合保存了trx_id到trx_t的关联，可以非常快速的定位到事务对象，这个优化带来的性能提升已经没那么明显了。

关于隐式锁更详细的信息，我们将在之后专门讲述“事务锁”的月报中再单独描述。

###事务并发控制

在MySQL5.7中，由于消除了大量临界资源的竞争，InnoDB只读查询的性能非常优化，几乎可以随着CPU线性扩展。但如果进入到读写混合的场景，就不可避免的使用到一些临界资源，例如事务、锁、日志等子系统。当竞争越激烈，就可能导致性能的下降。通常系统会有个吞吐量和响应时间最优的性能拐点。

InnoDB本身提供了并发控制机制，一种是语句级别的并发控制，另外一种是事务提交阶段的并发控制。

语句级别的并发通过参数`innodb_thread_concurrency`来控制，表示允许同时在InnoDB层活跃的并发SQL数。

每条SQL在进入InnoDB层进行操作之前都需要先递增全局计数，并为当前sql分配`innodb_concurrency_tickets`个ticket。也就是说，如果当前sql需要进出InnoDB层很多次（例如一个大查询需要扫描很多行数据时），`innodb_concurrency_tickets`次都可以自由进入InnoDB，无需判断`innodb_thread_concurrency`。当ticket用完时，就需要重新进入。当SQL执行完成后，会将ticket重置为0。

如果当前InnoDB层的并发度已满，用户线程就需要等待，目前的实现使用sleep一段时间的方式，sleep的时间是自适应的，但你可以通过参数`innodb_adaptive_max_sleep_delay`来设置一个最多大sleep事件，具体的算法参阅函数`srv_conc_enter_innodb_with_atomics`。

提到并发控制，另外一个不得不提的问题就是热点更新问题。事务在进入InnoDB层，准备更新一条数据，但发现行记录被其他线程锁住，这时候该线程会强制退出innodb并发控制，同时将自己suspend住，进入睡眠等待。如果有大量并发的更新同一条记录，就意味着大量线程进入InnoDB层，访问热点竞争资源锁系统，然后再退出。最终会呈现出大量线程在innodb中suspend住，相当于并发控制并没有达到降低临界资源争用的效果。早期我们对该问题的优化就是将线程从堵在innodb层，转移到堵在进入innodb层时的外部排队中，这样就不涉及到InnoDB的资源争用了。具体的实现是将statement级别的并发控制提升为事务级别的并发控制，因此这个方案的缺陷是对长事务不友好。

另外还有一些并发控制方案，例如线程池、水位限流、按pk排队等策略，我们的RDS MySQL也很早就支持了。如果你存在热点争用（例如秒杀场景），并且正在使用RDS MySQL，你可以去咨询售后如何使用这些特性。

除了语句级别的并发外，InnoDB也提供了提交阶段的并发控制，主要通过参数`innodb_commit_concurrency`来控制。该参数的默认值为0，表示不控制commit阶段的并发。在进入函数`innobase_commit`时，如果该参数被设置，且当前并发度超过，就需要等待。然而由于当前在默认配置下所有事务都走组提交(`ordered_commit`)，innodb层的提交大多数情况下只会有一个活跃线程。你只有关闭binlog或者关闭参数`binlog_order_commits`，这个参数设置才有意义。

###高优先级事务

MySQL5.7 实现了一种高优先级的事务调度方式。当事务处于高优先级模式时，它将永远不会被选作deadlock场景的牺牲者，拥有获得锁的最高优先级，并能kill掉阻塞它的的优先级事务。这个特性主要是为了支持官方开发的Group Replication Plugin套件，以保证事务总是能在所有的节点上提交。

**如何使用**
目前GA版本还没有提供公共接口来使用该功能，但代码实现都是完备的，如果想使用该功能，直接写一个设置变量的接口即可，非常简单。
在server层，每个THD上新增了两个变量来标识事务的优先级：
* `THD::tx_priority` 事务级别有效，当两个事务在innodb层冲突时，拥有更高值的事务将赢得锁
* `THD::thd_tx_priority` 线程级别有效，当该变量被设置时，选择该值作为事务优先级，否则选择tx_priority

**死锁检测**
在进行死锁检测时，需要对死锁的两个事务的优先级进行比较，低优先级的总是会被优先回滚掉，以保证高优先级的事务正常执行（`DeadlockChecker::check_and_resolve`）

**处理锁等待**
在对记录尝试加锁时，如果发现有别的事务和当前事务冲突（`lock_re_other_has_conflicting`），需要判断是否要加入到等待队列中（`RecLock::add_to_wait`）：
* 如果两个事务都设置了高优先级、但当前事务优先级较低，或者冲突的事务是一个后台进程开启的事务（例如dict_stat线程进行统计信息更新），则立刻失败该事务，并返回DB_DEADLOCK错误码

* 尝试将当前锁对象加入到等待队列中(`RecLock::enqueue_priority`)，高优先级的事务可以跳过锁等待队列(`RecLock::jump_queue`)，被跳过的事务需要被标记为异步回滚状态（`RecLock::mark_trx_for_rollback`），搜集到当前事务的`trx_t::hit_list`链表中。当阻塞当前事务的另外一个事务也处于等待状态、但等待另外一个不同的记录锁时，调用`rollback_blocking_trx`直接回滚掉，否则在进入锁等待之前再调用`trx_kill_blocking`依次回滚。

这里涉及到事务锁模块，本文不展开描述，下次专门在事务锁相关主题的月报讲述，你可以通过官方[WL#6835](http://dev.mysql.com/worklog/task/?id=6835) 获取更多关于高优先级事务的信息

 
###trx_t::flush_observer   

阅读代码时发现这个在5.7版本新加的变量，从它的命名可以看出，其应该和脏页flush相关。`flush_observer`可以认为是一个标记，当某种操作完成时，对于带这种标记的page(`buf_page_t::flush_observer`)，需要保证完全刷到磁盘上。

该变量主要解决早期5.7版本建索引耗时太久的[bug#74472](http://bugs.mysql.com/bug.php?id=74472)：为表增加索引的操作非常慢，即使表上没几条数据。原因是innodb在5.7版本修正了建索引的方式，采用自底向上的构建方式，并在创建索引的过程中关闭了redo，因此在做完加索引操作后，必须将其产生的脏页完全写到磁盘中，才能认为索引构建完毕，所以发起了一次完全的checkpoint，但如果buffer pool中存在大量脏页时，将会非常耗时。

为了解决这一问题，引入了`flush_observer`，在建索引之前创建一个`FlushObserver`并分配给事务对象(`trx_set_flush_observer`)，同时传递给`BtrBulk::m_flush_observer`。

在构建索引的过程中产生的脏页，通过`mtr_commit`将脏页转移到flush_list上时，顺便标记上flush_observer（`add_dirty_page_to_flush_list —> buf_flush_note_modification`）

当做完索引构建操作后，由于bulk load操作不记redo，需要保证DDL产生的所有脏页都写到磁盘，因此调用FlushObserver::flush，将脏页写盘（`buf_LRU_flush_or_remove_pages`）。在做完这一步后，才开始apply online ddl过程中产生的row log(`row_log_apply`)。

如果DDL被中断（例如session被kill），也需要调用`FlushObserver::flush`，将这些产生的脏页从内存移除掉，无需写盘。

###事务对象池

为了减少构建事务对象时的内存操作开销，尤其是短连接场景下的性能，InnoDB引入了一个池结构，可以很方便的分配和释放事务对象。实际上事务的事务锁对象也引用了池结构。

事务池对应的全局变量为trx_pools，初始化为：

```
     trx_pools = UT_NEW_NOKEY(trx_pools_t(MAX_TRX_BLOCK_SIZE));
```
trx_pools是操作trx pool的接口，类型为trx_pools_t，其定义如下：

```
     typedef Pool<trx_t, TrxFactory, TrxPoolLock> trx_pool_t;
     typedef PoolManager<trx_pool_t, TrxPoolManagerLock > trx_pools_t;
```

其中,trx_t表示事务对象类型，TrxFactory封装了事务的初始化和，TrxPoolLock封装了POOL锁的创建，销毁，加锁，解锁。
PoolManager封装了池的管理方法

这里涉及到多个类：
* Pool 及 PoolManager 是公共用的类
* TrxFactory 和 TrxPoolLock, TrxPoolManagerLock是trx pool私有的类。
* TrxFactory用于定义池中事务对象的初始化和销毁动作；
* TrxPoolLock用于定义每个池中对象的互斥锁操作；
* 由于POOL的管理结构支持多个POOL对象， TrxPoolManagerLock用于互斥操作增加POOL对象。支持多个POOL对象的目的是分拆单个POOL对象的锁开销，避免引入热点。因为从POOL中获取和返还对象，都是需要排他锁的。

相关类的关系如下图所示：
![5](http://img2.tbcdn.cn/L1/461/1/708d901508481bbb1564a4f3c942b924ba190dae.png)

##InnoDB  MVCC实现

InnoDB有两个非常重要的模块来实现MVCC，一个是undo日志，用于记录数据的变化轨迹，另外一个是Readview，用于判断该session对哪些数据可见，哪些不可见。实际上我们已经在之前的月报中介绍过这部分内容，这里再简要介绍下。

###事务视图ReadView

前面已经多次提到过ReadView，也就是事务视图，它用于控制数据的可见性，在InnoDB中，只有查询才需要通过Readview来控制可见性，对于DML等数据变更操作，如果操作了不可见的数据，则直接进入锁等待。

ReadView包含几个重要的变量：
*  `ReadView::id` 创建该视图的事务id
*  `ReadView::m_ids` 创建readview时，活跃的读写事务id数组，有序存储
* `ReadView::m_low_limit_id` 设置为当前最大事务id
* `ReadView::m_up_limit_id`  m_ids集合中的最小值，如果m_ids集合为空，表示当前没有活跃读写事务，则设置为当前最大事务id

很显然ReadView的创建需要在trx_sys->mutex的保护下进行，相当于拿到了当时的一个全局事务快照。基于上述变量，我们就可以判断数据页上的记录是否对当前事务可见了。

为了管理ReadView，MVCC子系统使用多个链表进行分配、维护、回收Readview
* `MVCC::m_free` 用于维护空闲的readview对象，初始化时创建1024个ReadView对象（`trx_sys_create`），当释放一个活跃的视图时，会将其加到该链表上，以便下次重用。
* `MVCC::m_views` 这里存储了两类视图，一类是当前活跃的视图，另一类是上次被关闭的只读事务视图。后者主要是为了减少视图分配开销。因为当系统的读占大多数时，如果在两次查询中间没有进行过任何读写操作，那我们就可以重用这个readview，而无需去持有`trx_sys->mutex`锁重新分配。
目前自动提交的只读事务或者RR级别下的只读都支持read view缓存，但目前版本还存在的问题是，在RC级别下不支持视图缓存，见[bug#79005](http://bugs.mysql.com/bug.php?id=79005)

另外purge系统在开始purge任务时，需去克隆`MVCC::m_views`链表上未被close的最老视图，并在本地视图中将该最老事务的事务id也加入到不可见的事务id集合`ReadView::m_ids`中(`MVCC::clone_oldest_view`)

###回滚段指针
回滚段undo是实现InnoDB MVCC的根基。每次修改聚集索引页上的记录时，变更之前的记录都会写到undo日志中。回滚段指针包括undo log所在的回滚段id、日志所在的page no、以及page内的偏移量，可以据此找到最近一次修改之前的undo记录，而每条Undo记录又能再次找到之前的变更。

当有可能undo被访问到时，Purge_sys将不会去清理undo log，如上所述，purge_sys只会去清理最老readview不会看到的事务。这意味着，如果你运行了一个长时间的查询SQL，或者以大于RC的隔离级别开启了一个事务视图但没有提交事务，Purge系统将一直无法前行，即使你的会话并不活跃。这时候undo日志将无法被及时回收，最直观的后果就是undo空间急剧膨胀。

关于undo这里不赘述，详细参阅[之前月报](http://mysql.taobao.org/monthly/2015/04/01/)

###可见性判断

如上所述，聚集索引的可见性判断和二级索引的可见性判断略有不同。因为二级索引记录并没有存储事务id信息，相应的，只是在数据页头存储了最近更新该page的trx_id。

对于聚集索引记录，当我们从btree获得一条记录后，先判断（`lock_clust_rec_cons_read_sees`）当前的readview是否满足该记录的可见性：

*  如果记录的trx_id小于`ReadView::m_up_limit_id`，则说明该事务在创建readview时已经提交了，肯定可见
*  如果记录的trx_id大于等于`ReadView::m_low_limit_id`，则说明该事务是创建readview之后开启的，肯定不可见。
*  当trx_id在`m_up_limit_id`和`m_low_limit_id`之间时，如果在`ReadView::m_ids`数组中，说明创建readview时该事务是活跃的，其做的变更对当前视图不可见，否则对该trx_id的变更可见。

如果基于上述判断，该数据变更不可见时，就尝试通过undo去构建老版本记录（`row_sel_build_prev_vers_for_mysql -->row_vers_build_for_consistent_read`），直到找到可见的记录，或者到达undo链表头都未找到。

注意当隔离级别设置为READ UNCOMMITTED时，不会去构建老版本。

如果我们查询得到的是一条二级索引记录：
* 首先将page头的trx_id和当前视图相比较：如果小于`ReadView::m_up_limit_id`，当前事务肯定可见；否则就需要去找到对应的聚集索引记录（`lock_sec_rec_cons_read_sees`）。
* 如果需要进一步判断，先根据ICP条件，检查是否该记录满足push down的条件，以减少回聚集索引的次数。
* 满足ICP条件，则需要查询聚集索引记录（`row_sel_get_clust_rec_for_mysql`），之后的判断就和上述聚集索引记录的判断一致了

在InnoDB中，只有读查询才会去构建ReadView视图，对于类似DML这样的数据更改，无需判断可见性，而是单纯的发现事务锁冲突，直接堵塞操作。

###隔离级别

然而在不同的隔离级别下，可见性的判断有很大的不同。

* READ-UNCOMMITTED
在该隔离级别下会读到未提交事务所产生的数据更改，这意味着可以读到脏数据，实际上你可以从函数`row_search_mvcc中`发现，当从btree读到一条记录后，如果隔离级别设置成READ-UNCOMMITTED，根本不会去检查可见性或是查看老版本。这意味着，即使在同一条SQL中，也可能读到不一致的数据。

* READ-COMMITTED
在该隔离级别下，可以在SQL级别做到一致性读，当事务中的SQL执行完成时，ReadView被立刻释放了，在执行下一条SQL时再重建ReadView。这意味着如果两次查询之间有别的事务提交了，是可以读到不一致的数据的。

* REPEATABLE-READ
可重复读和READ-COMMITTED的不同之处在于，当第一次创建ReadView后（例如事务内执行的第一条SEELCT语句），这个视图就会一直维持到事务结束。也就是说，在事务执行期间的可见性判断不会发生变化，从而实现了事务内的可重复读。

* SERIALIZABLE
序列化的隔离是最高等级的隔离级别，当一个事务在对某个表做记录变更操作时，另外一个查询操作就会被该操作堵塞住。同样的，如果某个只读事务开启并查询了某些记录，那么另外一个session对这些记录的更改操作是被堵塞的。内部的实现其实很简单：
    * 对InnoDB表级别加LOCK_IS锁，防止表结构变更操作
    * 对查询得到的记录加LOCK_S共享锁，这意味着在该隔离级别下，读操作不会互相阻塞。而数据变更操作通常会对记录加LOCK_X锁，和LOCK_S锁相冲突，InnoDB通过给查询加记录锁的方式来保证了序列化的隔离级别。

注意不同的隔离级别下，数据具有不同的隔离性，甚至事务锁的加锁策略也不尽相同，你需要根据自己实际的业务情况来进行选择。

###一个有趣的可见性问题

在READ-COMMITTED隔离级别下，我们考虑如下执行序列

```
Session 1：
mysql> CREATE TABLE t1 (c1 INT PRIMARY KEY, c2 INT, c3 INT, key(c2));
Query OK, 0 rows affected (0.00 sec)

mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> INSERT INTO t1 VALUES (1,2,3);
Query OK, 1 row affected (0.00 sec)

Session 2:
mysql> BEGIN;
Query OK, 0 rows affected (0.00 sec)

mysql> UPDATE t1 SET c3=c3+1 WHERE c3 = 3;    // 扫描聚集索引进行查询，记录不可见，但未被记录锁堵塞
Query OK, 0 rows affected (0.00 sec)
Rows matched: 0  Changed: 0  Warnings: 0

mysql> UPDATE t1 SET c3=c3+1 WHERE c2 = 2;   // 根据二级索引进行查询，记录不可见，且被记录锁堵塞
```

查询条件不同，但指向的确是同一条已插入未提交的记录，为什么会有两种不同的表现呢？ 这主要是不同索引在数据检索时的策略不同造成的。

实际上session2的第一条Update也为session1做了隐式锁转换，但是在返回到row_search_mvcc时，会走到如下判断：

```
Line 5312~5318,  row0sel.cc:
                        if (UNIV_LIKELY(prebuilt->row_read_type
                                        != ROW_READ_TRY_SEMI_CONSISTENT)
                            || unique_search
                            || index != clust_index) {

                                goto lock_wait_or_error;
                        }
```

* 对于第一条和第二条update， `prebuilt->row_read_type`值均为`ROW_READ_TRY_SEMI_CONSISTENT`，不满足第一个条件；
* 均不满足unique_search(通过pk，或uk作为where条件进行查询)
* 第一个使用的聚集索引，三个条件都不满足；而第二个update使用的二级索引，因此走`lock_wait_or_error`的逻辑，进入锁等待。

第一条update继续往下走，根据undo去构建老版本记录（`row_sel_build_committed_vers_for_mysql `），一条新插入的记录老版本就是空了，所以认为这条更新没有查询到目标记录，从而忽略了锁阻塞的逻辑。

如果使用pk或者二级索引作为where条件查询的话，都会走到锁等待条件。

推而广之，如果表上没有索引的话，那么对于任意插入的记录，更新操作都见不到插入的记录（但是会为插入操作创建记录锁）。

##InnoDB ACID

本小节针对ACID这四种数据库特性分别进行简单描述。

###Atomicity （原子性）
所谓原子性，就是一个事务要么全部完成变更，要么全部失败。如果在执行过程中失败，回滚操作需要保证“好像”数据库从没执行过这个事务一样。

从用户的角度来看，用户发起一个COMMIT语句，要保证事务肯定成功完成了；若发起ROLLBACK语句，则干净的回滚掉事务所有的变更。
从内部实现的角度看，InnoDB对事务过程中的数据变更总是维持了undo log，若用户想要回滚事务，能够通过Undo追溯最老版本的方式，将数据全部回滚回来。若用户需要提交事务，则将提交日志刷到磁盘。

###Consistency （一致性）
一致性指的是数据库需要总是保持一致的状态，即使实例崩溃了，也要能保证数据的一致性，包括内部数据存储的准确性，数据结构（例如btree）不被破坏。InnoDB通过doublewrite buffer 和crash recovery实现了这一点：前者保证数据页的准确性，后者保证恢复时能够将所有的变更apply到数据页上。如果崩溃恢复时存在还未提交的事务，那么根据XA规则提交或者回滚事务。 最终实例总能处于一致的状态。 

另外一种一致性指的是数据之间的约束不应该被事务所改变，例如外键约束。ySQL支持自动检查外键约束，或是做级联操作来保证数据完整性，但另外也提供了选项`foreign_key_checks`，如果您开启了这个选项，数据间的约束和一致性就会失效。有些情况下，数据的一致性还需要用户的业务逻辑来保证。

###Isolation （隔离性）
隔离性是指多个事务不可以对相同数据同时做修改，事务查看的数据要么就是修改之前的数据，要么就是修改之后的数据。InnoDB支持四种隔离级别，如上文所述，这里不再重复。

###Durability（持久性）
当一个事务完成了，它所做的变更应该持久化到磁盘上，永不丢失。这个特性除了和数据库系统相关外，还和你的硬件条件相关。 InnoDB给出了许多选项，你可以为了追求性能而弱化持久性，也可以为了完全的持久性而弱化性能。

和大多数DBMS一样，InnoDB也遵循WAL（Write-Ahead Logging）的原则，在写数据文件前，总是保证日志已经写到了磁盘上。通过Redo日志可以恢复出所有的数据页变更。

为了保证数据的正确性，Redo log和数据页都做了checksum校验，防止使用损坏的数据。目前5.7版本默认支持使用CRC32的数据校验算法。

为了解决半写的问题， 即写一半数据页时实例crash，这时候数据页是损坏的。InnoDB使用double  write buffer来解决这个问题，在写数据页到用户表空间之前，总是先持久化到double write buffer，这样即使没有完整写页，我们也可以从double write buffer中将其恢复出来。你可以通过innodb_doublewrite选项来开启或者关闭该特性。

InnoDB通过这种机制保证了数据和日志的准确性的。你可以将实例配置成事务提交时将redo日志fsync到磁盘（innodb_flush_log_at_trx_commit = 1），数据文件的FLUSH策略（innodb_flush_method）修改为0_DIRECT，以此来保证强持久化。你也可以选择更弱化的配置来保证实例的性能。
