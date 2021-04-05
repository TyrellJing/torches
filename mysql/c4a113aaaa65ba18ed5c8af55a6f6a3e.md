# Checkpoint技术

对MySQL的操作首先都是在缓冲池中完成的，例如一条DML语句，如果Update或Delete改变了页中的记录，那么此时缓冲池中的页版本比测盘的要新，页是脏的，数据库需要将新版本的页从缓冲池刷新到磁盘。

如果每一次页发生变化就将新的版本刷新到磁盘，那么这个开销将非常大。如果热点数据集中在某几个页中，数据库性能将变得很差。同时，如果在缓冲池将页的新版本刷新到磁盘时发生了宕机，那么数据就能恢复了。

为了减少频繁刷新脏页到磁盘引起的性能损耗，同时避免发生数据丢失的问题，当前事务数据库普遍采用了Write Ahead Log策略，即当事务提交时，先写重做日志，再修改页。当数据库宕机时，通过重做日志来完成数据恢复，满足事务持久性要求。

Checkpoint技术的目的主要解决了以下几个问题：

- 缩短数据库的恢复时间，当数据库发生宕机时，数据库不需要重做所有日志，因为Checkpoint之前的页都已经刷新回磁盘。故数据库只需对Checkpoint之后的重做日志进行恢复就可以了，缩短了恢复时间。

- 缓冲池不够用时，刷新脏页。根据LRU算法溢出最近最少使用的页，如果存在脏页，需要执行Checkpoint将脏页刷回磁盘。

- 重做日志不够用时，刷新脏页。当重做日志不够用时，需要循环覆盖部分重做日志，这时需要保证这部分日志被刷回磁盘。

InnoDB存储引擎通过LSN(Log Sequence Number)来标记版本。LSN是8字节的数字，每个页有LSN，重做日志有LSN，Checkpoint也有LSN。可以通过SHOW ENGINE INNODB STATUS来查看。

在InnoDB内部有两种Checkpoint：

- Sharp Checkpoint：在数据库关闭时将所有的脏页都刷新回磁盘，这是默认的工作方式，参数innodb_fast_shutdown=1。

- Fuzzy Checkpoint：在数据库运行时刷新部分脏页到磁盘。大致可以分为下面四种：

    - Master Thread Checkpoint：在Master Thread中发生的Checkpoint，差不多每秒或每十秒的速度从缓冲池的脏页列表中刷新一定比例的页回磁盘，这个过程是异步的，此时InnoDB存储引擎可以进行其他操作，用户查询线程不会阻塞。

    - FLUSH_LRU_LIST Checkpoint：InnoDB需要保证LRU列表中需要有差不多100个空闲页可供使用，如果没有100个可用的空闲页，那么将从LRU列表尾端移除，如果其中有脏页将进行Checkpoint。可以通过配置innodb_lru_scan_depth控制LRU列表中可用页数量，该值默认为1024。

    - Async/Sync Flush Checkpoint：重做日志不可用时，需要强制将一些页刷新到磁盘，此时脏页是从脏页列表中选取的。若此时已经写入重做日志的LSN计为redo_lsn，将已经刷新回磁盘最新页LSN计为checkpoint_lsn，则可定义

    ```
        checkpoint_age = redo_lsn - checkpoint_lsn
    ```
    再定义以下变量：

    ```
        async_water_mark = 75% * total_redo_log_file_size
        sync_water_mark  = 90% * total_redo_log_file_size
    ```
    若每个重做日志文件大小为1GB，并且定义了两个重做日志文件，则重做日志文件总大小为2GB，那么async_water_mark=1.5GB，sync_water_mark=1.8GB.

    1. 当checkpoint_age < async_water_mark时，不需要刷新任何脏页到磁盘

    2. 当async_water_mark < checkpoint_age < sync_water_mark时触发Async Flush，从Flush列表中刷新足够的脏页到磁盘，使得刷新后满足checkpoint_age < async_water_mark

    3. 当checkpoint_age > sync_water_mark时，这种情况一般是重做日志过小，并且在进行类似LOAD DATA的BULK INSERT操作。此时触发Sync Flush操作，从Flush列表中刷新足够多的脏页回磁盘，使得刷新后满足checkpoint_age < async_water_mark。

    上述两种Flush引发的Checkpoint有单独的Page Cleaner Thread完成，以免阻塞用户查询线程。

    - Dirty Page too much Checkpoint：脏页数量太多时，导致InnoDB存储引擎强制进行Checkpoint，其目的总的来说还是为了保证缓冲池中有足够可用的页。可由参数innodb_max_dirty_pages_pct控制，该参数默认值为90，即当脏页数占90%时进行Checkpoint。

