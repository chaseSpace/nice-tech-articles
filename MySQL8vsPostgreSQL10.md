## MySQL 8 vs PostgreSQL 10的较量

[原文链接🔗](https://medium.com/hackernoon/showdown-mysql-8-vs-postgresql-10-3fe23be5c19e)


现在[MySQL 8](https://mysqlserverteam.com/whats-new-in-mysql-8-0-generally-available/)和[PostgreSQL 10](https://www.postgresql.org/about/news/1786/)已经发布，是时候重新审视这两个主要的**开源关系数据库**是如何如何彼此竞争的了。

在这些版本之前，人们普遍认为，虽然Postgres在功能及其渊源方面更胜一筹，但MySQL更擅长大规模并发读/写操作测试。

但是随着最新版本的发布，两者之间的差距已大大缩小。

## 功能比较

让我们看一下我们都喜欢讨论的“新颖”功能。

![图片发布](https://miro.medium.com/max/60/1*RM4sMrIn4L7bcPywZWJRuw.png?q=20)

![图片发布](https://miro.medium.com/max/760/1*RM4sMrIn4L7bcPywZWJRuw.png)

过去经常说MySQL最适合在线事务，而PostgreSQL最适合分析过程。但现在不再是这个局面了。

通用表表达式（CTE）和窗口函数是选择PostgreSQL的主要原因。通过引用同一个表中的`boss_id`来递归地遍历一张`employees`表，或者在一个排好序的结果中找到一个中值（或50%），这在MySQL中同样不再是问题。

在PostgreSQL上的复制操作缺乏配置灵活性，这也就是[Uber转向MySQL](https://eng.uber.com/mysql-migration/)的原因。但是现在PostgreSQL有了逻辑复制的特性，就可以通过创建一个新版本的Postgres并切换到它来实现零停机时间升级。在截断大型时序事件表中陈旧分区的场景下也变的容易得多。

就这些功能而言，两个数据库现在保持一致。

## 区别在哪里？

现在，我们只剩下一个问题——为什么要选择其中一个而不是另一个呢？

**生态系统**是其中一个因素。MySQL拥有一个充满活力的生态系统，其中包含MariaDB，Percona，Galera等等，以及InnoDB以外的其他存储引擎，但这也可能令人困惑。Postgres被高端应用场景所限制，但是随着最新版本引入的新功能，这个情况将会改变。

**治理**是另一个因素。每个人都在担心Oracle（或最初为SUN）收购MySQL时，他们会毁了该产品，但就过去十年里来看，情况并非如此。实际上，在被收购之后，Mysql的发展反而加速了。而Postgres在工作治理和协作社区方面则拥有悠久的历史。

**基础架构**不会经常改变，虽然最近没有对这方面的详细讨论，但这也是值得考虑的。



下面是对二者之间特性的对比：

![图片发布](https://miro.medium.com/max/60/1*LtiA09xS90-5qbLdHquGiA.png?q=20)

![图片发布](https://miro.medium.com/max/760/1*LtiA09xS90-5qbLdHquGiA.png)

### 进程与线程

由于[Postgres通过fork子进程](https://www.postgresql.org/docs/10/static/tutorial-arch.html)来建立连接，可能需要[高达每连接10 MB](https://www.citusdata.com/blog/2017/05/10/scaling-connections-in-postgres/)。与MySQL的“每次连接创建一个独立线程服务”模型相比，前者内存的压力更大，后者在64位平台上，[线程](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_stack)的默认[堆栈大小为](https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_thread_stack)256KB。（当然，采用线程局部排序缓冲区等方式可以使得此开销降低，即使说起来可以忽略不计，但事实上还是会有所消耗。）

尽管通过[写时复制](https://en.wikipedia.org/wiki/Copy-on-write)的方式可以让子进程保留一些父进程共享的，不可变的内存状态，但是当有1,000个以上的并发连接时，基于进程的体系结构——Postgres的基本开销负担会有所增加，并且它可能是影响性能弹性的最关键的因素之一。

也就是说，如果您在30台服务器上运行Rails应用程序，其中每台服务器拥有16个CPU内核和32个Unicorn worker，那么可以有960个连接。在所有应用程序中，可能只有不到0.1％会达到这个规模，但这是需要牢记的。

### 聚簇索引与堆表

[聚簇索引](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/clustered-and-nonclustered-indexes-described?view=sql-server-2017)的特点：表数据是和主键一起存储的（叶节点存放一整行记录），直接嵌入B树结构内。（非聚簇索引）表数据和索引是分开存储。



使用聚集索引，当通过主键查找记录时，单个I / O会检索整行，而非聚集索引始终需要通过引用至少两个I / O的方式。由于外键引用和join的使用将触发主键查找，因此可能对性能的影响很大，而且绝大多数查询都是这样。

聚簇索引的理论缺点是，在使用二级索引进行查询时需要遍历两次树节点，因为首先遍历二级索引，然后遍历聚簇索引（也是一棵树）。

但是根据现在的[惯例](https://en.wikipedia.org/wiki/Convention_over_configuration)，我们会将自动递增整数(auto-increment integer)作为主键¹（称为[代理键](https://en.wikipedia.org/wiki/Surrogate_key) 没有业务含义）我们[几乎总是希望拥有聚簇索引](https://www.mssqltips.com/sqlservertutorial/3209/make-sure-all-tables-have-a-clustered-index-defined/)。如果需要频繁的执行操作`ORDER BY id`来检索最新（或最旧）的N条记录，那就更应该如此，我认为这适用于大多数情况。

> [1]顺便提一下，UUID作为主键是一个糟糕的主意-完全是采用密码随机的方式来 **避免** 局部性的产生，因此会降低性能。

Postgres不支持聚集索引，而MySQL（InnoDB）不支持非聚集索引。但是，无论哪种方式，在有大量内存的情况下，差异应该很小。

### 页面结构和压缩

Postgres和MySQL都具有基于页的物理存储。（8KB和16KB）

![图片发布](https://miro.medium.com/max/60/1*5t5c-FGHhdtwz1z2cmeOEQ.png?q=20)

![图片发布](https://miro.medium.com/max/1509/1*5t5c-FGHhdtwz1z2cmeOEQ.png)

[PostgreSQL物理存储简介](http://rachbelaid.com/introduction-to-postgres-physical-storage/)

在PostgreSQL上，页面结构看起来像上边的图像。

它包含一些header，我们在这里先不进行介绍，但是这里面包含有关页面的元数据。Item后的项目是一个数组标识符，由`(offset, length) `指向元组或数据行的数据对组成。请记住，在Postgres中，可以通过这种方式将同一记录的多个版本存储在同一页面中。

![图片发布](https://miro.medium.com/max/60/1*UKTHGq1eJEP4xCQN9GJUgg.jpeg?q=20)

![图片发布](https://miro.medium.com/max/1298/1*UKTHGq1eJEP4xCQN9GJUgg.jpeg)

MySQL(InnoDB)的表空间结构与Oracle的表空间结构相似，它具有段，范围，页和行的多个结构层。

它还为UNDO提供了一个单独的部分，称为“回滚部分”。与Postgres不同，MySQL将在同一区域保留同一记录的多个版本。

在这两个的数据库中，一行记录必须能够被填入一个单独的页中，这意味着一行记录必须小于8KB。（在Mysql中一个数据页至少要包含2行记录，巧合的是16KB / 2 = 8KB）

![图片发布](https://miro.medium.com/max/60/1*KDibjX9OFPupE-OMybsTQg.png?q=20)

![图片发布](https://miro.medium.com/max/838/1*KDibjX9OFPupE-OMybsTQg.png)

那么，当在列中有一个大型JSON对象时会发生什么事情呢？

Postgres使用[TOAST](https://wiki.postgresql.org/wiki/TOAST)（**超尺寸属性存储技术**，主要用于存储一个大字段的值）。当选择了行和列后，大对象会被分离出来。换句话说，这一大块对象不会占用您宝贵的缓存。Postgres还支持对已经TOAST过的对象进行压缩。

由于高端SSD存储供应商[Fusion-io](https://en.wikipedia.org/wiki/Fusion-io)的贡献，MySQL拥有一种叫做“[透明页压缩”](https://dev.mysql.com/doc/refman/8.0/en/innodb-page-compression.html)的更高级功能。它是专门为适配SSD而设计的，固态硬盘的写入量与设备的寿命直接相关。

MySQL上的压缩不经适用于页面外的大对象，还适用于所有页面。它是通过在[稀疏文件中](https://en.wikipedia.org/wiki/Sparse_file)使用`hole punching`技术来实现的，这种技术在现代文件系统（例如[ext4](https://en.wikipedia.org/wiki/Ext4)或[btrfs）](https://en.wikipedia.org/wiki/Btrfs)上都支持。

更多相关详细信息，请参阅：[FusionIO上的新MariaDB页面压缩显着提高性能](https://mariadb.org/significant-performance-boost-with-new-mariadb-page-compression-on-fusionio/)

### 更新带来的开销

我们经常会忽视另一个功能，但它对性能有重大影响，并且可能是饱含争议的主题，它是UPDATEs。

这是Uber放弃Postgres的另一个原因，这激起了许多Postgres拥护者的反驳。

- [MySQL可能适合Uber，但不适合您](https://dzone.com/articles/on-ubers-choice-of-databases)
- [PostgreSQL对Uber的回应（PDF）](http://thebuild.com/presentations/uber-perconalive-2017.pdf)

两者都是[MVCC](https://en.wikipedia.org/wiki/Multiversion_concurrency_control)数据库，因为[隔离](https://en.wikipedia.org/wiki/Isolation_(database_systems))保留了数据的多个版本。

为此，Postgres将旧数据保留在堆中直到VACUUMed，而MySQL将旧数据移动到回滚段。

在Postgres上，当您尝试更新操作时，必须复制整行以及指向该行的索引条目。部分原因是因为Postgres不支持聚集索引，因此从索引引用的行的物理位置不会被逻辑键抽象出来。



为了解决这个问题，Postgres使用[Heap Only Tuples (HOT)](http://www.interdb.jp/pg/pgsql07.html)尽可能不更新索引。但是，如果更新足够频繁（或者如果一个元组很大），则元组的历史记录很容易从8KB的页面大小中溢出，跨多个页面并限制了功能的有效性。修剪/或碎片整理的时间取决于试探。另外，将[fillfactor](https://www.postgresql.org/docs/10/static/sql-createtable.html#SQL-CREATETABLE-STORAGE-PARAMETERS)设置为小于100则会降低空间效率—这是在表创建时就不必担心的硬性折衷。



这个限制甚至愈发深了。由于索引元组中没有有关事务的任何信息，因此直到9.2以前一直不可能支持[Index-Only Scans](https://use-the-index-luke.com/sql/clustering/index-only-scan-covering-index)。它是所有主要数据库（包括MySQL，Oracle，IBM DB2和Microsoft SQL Server）支持的最古老，最重要的优化方法之一。但是即使是使用最新版本，当有大量的UPDATE设置[Visibility Map中的](http://www.interdb.jp/pg/pgsql06.html#_6.2.)更改位时，Postgres也不能完全支持`Index Only Scans`，而在不需要时经常选择Seq扫描。

在MySQL中，通常是原地更新，旧行数据存放在称为回滚段中。结果是您不需要VACUUM，提交会非常快，而回滚相对较慢，这对于大多数使用场景来说是一个较好的折衷方案。

它也[足够聪明，](https://www.percona.com/blog/2011/01/12/innodb-undo-segment-siz-and-transaction-isolation/)可以尽快清除历史记录。如果事务的隔离级别设置为**READ-COMMITTED**或更低，则在语句完成时清除历史记录。

事务记录的大小不会影响到主页。碎片也不是个问题。因此，MySQL的整体性能更好，更可预测。

### 垃圾收集

Postgres上的VACUUM花费十分巨大，因为它可以在主堆区域中工作，从而造成直接的资源争用。感觉就像编程语言中的垃圾回收一样，它会妨碍您并会让您突然暂停。

具有数十亿条记录的表[Configuring autovacuum](https://www.citusdata.com/blog/2016/11/04/autovacuum-not-the-enemy/)仍然是一个很大的挑战。

对MySQL的清除也可能很繁重，但是由于它在单独的回滚段中使用专用线程运行，因此它不会以任何方式对并发读取产生不利的影响。即使使用[默认设置](https://dev.mysql.com/doc/refman/8.0/en/innodb-improved-purge-scheduling.html)，膨胀的回滚段也不太可能会降低性能。

一个拥有数十亿条记录的大表不会导致MySQL的历史记录膨胀，并且诸如存储文件大小和查询性能之类的事也是可以预测并且稳定下来的。

### 日志和复制

Postgres 拥有被称作 [Write Ahead Log（WAL）](http://www.interdb.jp/pg/pgsql09.html)的单信源事务历史。它也用于复制，称为逻辑复制的新功能可以将二进制内容快速解码为更易理解的逻辑语句，从而可以对数据进行精细的控制。

MySQL维护两个单独的日志：1.用于崩溃恢复的InnoDB特定[重做日志](https://dev.mysql.com/doc/refman/8.0/en/innodb-redo-log.html)，以及2.用于复制和增量备份的[二进制日志](https://dev.mysql.com/doc/refman/8.0/en/binary-log.html)。

InnoDB 上的重做日志与 Oracle 一致，它是一个自治的[循环缓冲区](https://en.wikipedia.org/wiki/Circular_buffer)，不会随着时间的推移而增长，只能在启动时以固定大小创建。这种设计可确保在物理设备上保留连续的连续区域，从而提高性能。redolog越大，性能越好，但这要以崩溃恢复时间为代价。

在Postgres中添加了新的复制功能后，我觉得Mysql与Postgres打成了平手。

## TL; DR

令人惊讶的结果是，事实上，正确的仍然是大多数。MySQL最适合在线事务，而PostgreSQL最适合仅用于追加分析过程，例如数据仓库²。

> [2]当我说Postgres非常适合分析时，我是说真的。如果您不了解[TimescaleDB](https://blog.timescale.com/timescaledb-vs-6a696248104e)，它是PostgreSQL之上的再封装应用，可以每秒插入100万条记录，每台服务器又1000亿行。这也太强了，难怪[亚马逊](https://docs.aws.amazon.com/redshift/latest/dg/c_redshift-and-postgres-sql.html)为什么[选择PostgreSQL作为Redshift的基础](https://docs.aws.amazon.com/redshift/latest/dg/c_redshift-and-postgres-sql.html)。

正如我们在本文中看到的，Postgres的绝大多数复杂性源于其仅适合追加，过度冗余的堆体系结构。

在后续版本中，Postgres可能需要对其存储引擎进行重大改进。如果您不相信我的话-[官方Wiki上](https://wiki.postgresql.org/wiki/Future_of_storage)已经[讨论了](https://wiki.postgresql.org/wiki/Future_of_storage)它，这意味着是时候从InnoDB那里吸取一些优点了。



我们一次又一次地说MySQL正在追赶Postgres，但是这一次，时代变了。