---
title: '[Mysql]两阶段提交和崩溃恢复'
tags:
  - Mysql
slug: 2934732838
date: 2020-06-05 11:15:08
---
# 两阶段提交
innodb和bin log需要进行两阶段提交(Two-Phase Commit Protocol，2PC)，目的是为了保证日志一致性，要么都存在，要么都不存在。

要说明的是，两阶段提交不止redo log和binlog有关，undo log也是参与者。

XID是事务根据事务第一条query生成的id，通过XID将redo log和binlog关联起来。在上一篇的binlog日志里也能看到XID的身影。

两阶段提交，其实就是innodb prepare，写binlog，innodb commit；

具体是，
1. prepare阶段，redo log写入根据事务提交时的刷盘策略决定是否刷盘，然后将log segment(不是回滚段！！！)标为`TRX_UNDO_PREPARED`；binlog不做任何事
2. commit阶段，binlog写入binlog日志；接着innodb将修改undo log segment状态，并在redo log写入一个commit log。

如果没有两阶段提交，redo log和binlog各写各的，中间发生崩溃，就会出现不一致的情况。

1. 先写binlog，再写redo log：如果redo log没写成功，就会造成崩溃恢复的时候，主库没有、从库有的情况，主从不一致。
2. 先写redo log，再写binlog：如果binlog没写成功，就会造成崩溃恢复的时候，主库有、从库没有的情况，主从不一致。

说白了，为了保证事务，总是得做一些冗余的操作，例如tcp多次握手挥手，都是通过冗余操作来保证两个业务之间一致。
## redo log和binlog顺序不一致的问题
两阶段的提交不只如此，如果只是像上面那样，还会出现主从不一致的情况。
### 热备问题

    T1 (--prepare--binlog[pos100]--------------------------------------------commit)
    T2 (-----prepare-----binlog[pos200]----------commit)
    T3 (--------prepare-------binlog[pos300]------commit)

    online-backup(----------------------------------------------backup------------)

假设3个事务如上，那么

    redo log prepare的顺序：T1 --> T2 --> T3
    binlog的写入顺序：      T1 --> T2 --> T3
    redo log commit的顺序： T2 --> T3 --> T1

可以看到redo log提交的顺序和bin log不一致了，这是不允许的，会导致主从不一致。

online-backup表示热备，因为从库在建立的时候需要对主库进行一次备份。当T2，T3提交后，这时热备来读位置，读到最后一个提交的事务T3，由于这个阶段不是读binlog的，所以T1没有被复制到，接下来的binlog复制从T3开始，所以会漏掉T1的数据。
### 复制问题
《MySQL5.7 核心技术揭秘：MySQL Group commit》（见参考）举了一个例子，就是有一行数据，x=1，y=1
T1:x=y+1,y=x+1;
T2:y=x+1,x=y+1;

文章说这两个事务颠倒执行出来的结果不一样。**但是在我看来**，这两个事务是没法颠倒的。事务为了防止**回滚覆盖**，对一条记录加X锁修改后，只有事务提交之后才能释放X锁，所以这两个事务是没办法同时处于2PC的，肯定是T1 2PC完，T2才开始2PC。才疏学浅，可能理解有误，欢迎指出。

所以早期的mysql用`prepare_commit_mutex`锁来发起2PC保证顺序，这在高并发下是很耗费性能的。一个事务要获取锁才能发起prepare，知道commit之后才释放锁。除了锁竞争，另一方面，`sync_binlog=1`的情况下，每次2PC需要刷盘binlog刷盘，这不仅增大了磁盘压力，也延长了占有锁的时长。

于是mysql5.6引入了组提交。
## 组提交
组提交是通过一个机制保证binlog顺序和commit顺序一致。

加入组提交之后，2PC的过程稍微变了。在将commit阶段细分，保证commit顺序和写binlog一致。

这个过程每个阶段都用了一个队列来存储，先到的线程是list的leader，后到的加入链表成为follower，

1. prepare阶段，事务获取`prepare_commit_mutex`，然后刷盘，设置prepare状态，然后释放锁。
2. commit阶段分成三个阶段：
   1. Flush stage: leader获得`Lock_log mutex`锁，将队列里的binlog写入文件缓冲
   2. Sync stage:leader释放`Lock_log mutex`，持有`Lock_sync mutex`, 如果sync_binlog为1，进行sync操作
   3. Commit stage: leader释放`Lock_sync mutex`，持有`Lock_commit mutex`，遍历队列，逐一进行commit。

每个阶段的队列长度不是一致的，可能Flush阶段的leader会在Sync阶段追加进前一个队列，成为follower，但是follower永远是follower。

网上的这个图很形象。
![](/images/20200605174715.png)

在Sync stage阶段，有两个参数可以影响组提交：
* `binlog_group_commit_sync_delay=N`:这个参数表明在Sync stage等待多少μs后可以刷盘，等的越久，就越可能合并后来的队列，一次刷更多日志，但是相应的，事务响应就变慢。
* `binlog_group_commit_sync_no_delay_count=N`:当队列的事务个数达到N，就进行刷盘。

组提交优点：
* 将本该串行的过程变成可并行的过程
* 虽然`prepare_commit_mutex`没有去除，但是占用的时间变短了，变成1/4
* 通过队列保证顺序一致
* 合并刷盘

# 崩溃恢复
## 未开启binlog
从redo log读到last checkpoint lsn，然后从这个位置开始重新应用redo log，不管是提交还是未提交状态。由于undo log会记录成redo log，所以构造出undo log之后，可以通过undo log回滚未提交的事务。
## 开启binlog
先和上面执行一样的逻辑，由于多了binlog，为了和从库保证一致，需要提取最后一个binlog文件的XID，接着检查处于prepare状态的redo log，如果redo log的XID不在binlog里，则回滚，如果在，提交redo log。

# 参考
1. [《MySQL5.7 核心技术揭秘：MySQL Group commit》](http://keithlan.github.io/2018/07/24/mysql_group_commit/)
