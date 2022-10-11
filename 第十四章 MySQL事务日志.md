# 第十四章 MySQL事务日志

事务有4种特性：原子性、一致性、隔离性和持久性；那么事务的四种特性到底是基于什么机制实现呢？

- 事务的隔离性由 锁机制 实现
- 而事务的原子性、一致性和持久性由事务的 redo 日志和undo 日志来保证
  - REDO LOG 称为 重做日志 ，提供再写入操作，恢复提交事务修改的页操作，用来保证事务的持 久性
  - UNDO LOG 称为 回滚日志 ，回滚行记录到某个特定版本，用来保证事务的原子性、一致性
- 总的来说，InnoDB存储引擎的原子性是通过`undo log`来保证，事务的持久性是通过`redo log`来实现的，事务的隔离性是通过`读写锁+MVCC机制`来实现的

## 14.1 redo log

> MySQL操作数据是在内存中完成的，然后再把内存中的数据页写入到磁盘中；
>
> 如果每次修改一条数据，就把整个内存页数据刷新到磁盘是非常浪费的，并且由于一个事务可能包含了多个执行语句，而执行语句对应的数据可能分散在不同的数据页，这样写磁盘就是多次随机IO操作，性能是非常低下的

但每一条数据的修改，都会记录一条`redo log`的记录，同样`redo log`也有自己的缓冲区存放数据修改的记录。当每个事务提交时，就会把缓存区中的记录刷新到磁盘中，同时由于磁盘中redo log的写入是顺序IO，所以效率也很高；变相来说，<font color="green">redo log实现了内存页数据刷新到磁盘从随机IO变成了顺序IO</font>，当然`Buffer Pool`本身在刷新数据到磁盘中可能还是随机IO

举例：

我们只是想让已经提交了的事务对数据库中数据所做的修改永久生效，即使后来系 统崩溃，在重启后也能把这种修改恢复出来。所以我们其实没有必要在每次事务提交时就把该事务在内 存中修改过的全部页面刷新到磁盘，`只需要把 修改 了哪些东西 记录一下 就好`。比如，某个事务将系统 表空间中 第10号 页面中偏移量为 100 处的那个字节的值 1 改成 2 。我们只需要记录一下：将第0号表 空间的10号页面的偏移量为100处的值更新为 2

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220920100356.png)

### redo log 优点

- redo日志降低了刷盘频率
- redo日志占用的空间非常小（存储表空间ID、页号、偏移量以及需要更新的值所需的存储空间是很小的）

### redo log 特点

- redo日志是顺序写入磁盘的（在执行事务的过程中，每执行一条语句，就可能产生若干条redo日志，这些日志是按照产生的顺序写入磁盘的，也就是使用顺序IO）
- 事务执行过程中，redo log不断记录

### redo log 组成

Redo log可以简单分为以下两个部分：

- 重做日志的缓冲 (redo log buffer) ，保存在内存中，是易失的

  - redo log buffer 大小，默认 16M ，最大值是4096M，最小值为1M

  - 查看

    ```mysql
    show variables like '%innodb_log_buffer_size%';
    ```

- 重做日志文件 (redo log file) ，保存在硬盘中，是持久的

### redo的整体流程

以一个更新事务为例，redo log 流转过程，如下图所示：

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220920101341.png)

第1步：先将原始数据从磁盘中读入内存中来，修改数据的内存拷贝 

第2步：生成一条重做日志并写入redo log buffer，记录的是数据被修改后的值 

第3步：当事务commit时，将redo log buffer中的内容刷新到 redo log file，对 redo log file采用追加 写的方式 

第4步：定期将内存中修改的数据刷新到磁盘中

###  redo log的刷盘策略

redo log的写入并不是直接写入磁盘的，InnoDB引擎会在写redo log的时候先写redo log buffer，之后以 一 定的频率 刷入到真正的redo log file 中。这里的一定频率怎么看待呢？

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220920101634.png)

注意，redo log buffer刷盘到redo log file的过程并不是真正的刷到磁盘中去，<font color="gree">只是刷入到 文件系统缓存 （page cache）中去</font>（这是现代操作系统为了提高文件写入效率做的一个优化），真正的写入会交给系 统自己来决定（比如page cache足够大了）。那么对于InnoDB来说就存在一个问题，如果交给系统来同 步，同样如果系统宕机，那么数据也丢失了（虽然整个系统宕机的概率还是比较小的）

针对这种情况，InnoDB给出 innodb_flush_log_at_trx_commit 参数，该参数控制 commit提交事务 时，如何将 redo log buffer 中的日志刷新到 redo log file 中。它支持三种策略：

- 设置为0 ：表示每次事务提交时不进行刷盘操作。（系统默认master thread每隔1s进行一次重做日 志的同步）

  ![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220920102410.png)

- 设置为1 ：表示每次事务提交时都将进行同步，刷盘操作（ 默认值 ）

  ![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220920102336.png)

- 设置为2 ：表示每次事务提交时都只把 redo log buffer 内容写入 page cache，不进行同步。由操作系统自己决定什么时候同步到磁盘文件

  ![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20220920102511.png)

### redo log buffer（未完成）

### redo log file（未完成）

#### 相关参数设置

- innodb_log_group_home_dir ：指定 redo log 文件组所在的路径，默认值为 ./ ，表示在数据库 的数据目录下。MySQL的默认数据目录（ var/lib/mysql ）下`默认有两个名为 ib_logfile0 和 ib_logfile1 的文件`，log buffer中的日志默认情况下就是刷新到这两个磁盘文件中。此redo日志 文件位置还可以修改
- innodb_log_files_in_group：指明redo log file的个数，命名方式如：ib_logfile0，iblogfile1... iblogfilen。默认2个，最大100个
- innodb_flush_log_at_trx_commit：控制 redo log 刷新到磁盘的策略，默认为1
- innodb_log_file_size：单个 redo log 文件设置大小，默认值为 48M 。最大值为512G，注意最大值 指的是整个 redo log 系列文件之和，即（innodb_log_files_in_group * innodb_log_file_size ）不能大 于最大值512G

#### 日志文件组

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20221001190124.png)

总共的redo日志文件大小其实就是： innodb_log_file_size × innodb_log_files_in_group 

采用循环使用的方式向redo日志文件组里写数据的话，会导致后写入的redo日志覆盖掉前边写的redo日志？当然！所以InnoDB的设计者提出了checkpoint的概念

## 14.2 Undo log

redo log是事务持久性的保证，<font color="pink">undo log是事务原子性的保证</font>。在事务中 更新数据 的 前置操作 其实是要 先写入一个 undo log

undo log是一种用于撤销回退的日志，在事务没提交之前，MySQL会先记录更新前的数据到 undo log日志文件里面，当事务回滚时或者数据库崩溃时，可以利用 undo log来进行回退

### 如何理解Undo日志

事务需要保证 原子性 ，也就是事务中的操作要么全部完成，要么什么也不做。但有时候事务执行到一半 会出现一些情况，比如：

- 情况一：事务执行过程中可能遇到各种错误，比如 服务器本身的错误 ， 操作系统错误 ，甚至是突 然 断电 导致的错误
- 情况二：程序员可以在事务执行过程中手动输入 ROLLBACK 语句结束当前事务的执行

以上情况出现，我们需要把数据改回原先的样子，这个过程称之为 回滚 ，这样就可以造成一个假象：这 个事务看起来什么都没做，所以符合 原子性 要求

### Undo日志的作用

1. 提供回滚操作【undo log实现事务的原子性】
2. 提供多版本控制(MVCC)【undo log实现多版本并发控制（MVCC）】

### Undo的存储结构

#### 回滚段与undo页

InnoDB对undo log的管理采用段的方式，也就是 <font color="pink">回滚段（rollback segment）</font> 。每个回滚段记录了 1024 个 undo log segment ，而在每个undo log segment段中进行 undo页 的申请

#### 回滚段与事务

1. 每个事务只会使用一个回滚段，一个回滚段在同一时刻可能会服务于多个事务
2. 当一个事务开始的时候，会制定一个回滚段，在事务进行的过程中，当数据被修改时，原始的数 据会被复制到回滚段
3. 在回滚段中，事务会不断填充盘区，直到事务结束或所有的空间被用完。如果当前的盘区不够 用，事务会在段中请求扩展下一个盘区，如果所有已分配的盘区都被用完，事务会覆盖最初的盘 区或者在回滚段允许的情况下扩展新的盘区来使用
4. 回滚段存在于undo表空间中，在数据库中可以存在多个undo表空间，但同一时刻只能使用一个 undo表空间
5. 当事务提交时，InnoDB存储引擎会做以下两件事情：
   - 将undo log放入列表中，以供之后的purge操作
   - 判断undo log所在的页是否可以重用，若可以分配给下个事务使用

#### 回滚段中的数据分类

1. 未提交的回滚数据(uncommitted undo information)
2. 已经提交但未过期的回滚数据(committed undo information)
3. 事务已经提交并过期的数据(expired undo information)

#### undo的类型

在InnoDB存储引擎中，undo log分为：insert undo log ; update undo log

#### undo log的生命周期

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20221001195008.png)

## 14.3 二者的区别

![](https://memory.xiaomage13148.xyz/typeroImage/QQ%E6%88%AA%E5%9B%BE20221001195114.png)

- undo log是逻辑日志，对事务回滚时，只是将数据库逻辑地恢复到原来的样子
- redo log是物理日志，记录的是数据页的物理变化，undo log不是redo log的逆过程





