# The Slippery Concept of a Transaction

## The Meaning of ACID

### Atomicity
很熟悉了，无须过多解释
### Consistency
这个词有点被滥用了，在很多场景意思都不同，书中举例了四种场景

在ACID中，consistency含义是，在应用层面有一些不变量，一定要保持其状态不变

因为是应用层面的概念，所以需要在应用代码逻辑中保证。   如果应用里写入DB打破invariant的数据 ，DB也无能为力
> Thus, the letter C doesn’t really belong in ACID.
### Isolation
并发场景（race condition）的概念。

经典教材也会把isolation称为 _serializability_，即每个transaction都可以假定他们是唯一正在运行的transaction，不会被其他transaction影响

实际应用中，极少有DB实现完全的isolation，因为对DB性能影响很大

### Durability
含义：不丢数据

- single node DB：保存到了disk or SSD（相对稳定的存储介质）。以及，存有write ahead log
- replicated DB：`data has been successfully copied to some number of nodes`。transaction需要在replication完成后才commit

不存在完美的durability。 比如极端情况下，所有的存储介质都挂了，且不可恢复

## Single-Object and Multi-Object Operations

### Single-obejct writes
single object的操作也是需要atomicity和isolation的，否则会遇到以下情况
- 较大的object write了一半网络断了
- overwrite遇到了断电。覆盖一半，还是保留old object？
- 并发场景，write没执行完的时候，read是否能读到中间态的write新数据？

### The need for multi-object transactions

Do we need multi-object transactions at all? 只有single object的transaction保障够吗？或许一些简单的场景是ok的，但是以下例子表明了multi-obejct transaction的必要性

- foreign key in relational DB
- verticle - edge - verticle in graph DB
- 文档型数据库，因为缺乏join功能，促进了denormalization（In computing, denormalization is the process of attempting to optimize the read performance of a database by adding redundant data or by grouping data）的使用。有了denormalized data，原数据和denormalized data就必要一起更新
- secondary index，需要和源数据、primary index一起更新

### Handling errors and aborts

一些未实现transaction的DB（比如使用了leaderless replication类型的DB），无法在error的时候自动回滚。 需要应用层自己处理

一些ORM框架（Rails’s ActiveRecord and Django）在transaction abort后，未实现自动retry的能力

但即使做了retry，一些情况下仍然无法最终达到效果。列举如下：
- 网络失败，导致trasaction实际成功但是client认为失败，发起不必要的retry。可能导致数据重复
- 如果error源于服务过载，继续重试只会让情况更糟。  需要控制重试的次数（比如使用exponential backoff算法），或者将overload导致的失败和其他类型的失败区分开，区分处理
- 短暂的error，重试或许可以成功。 但是由于一些明确逻辑原因的失败，再重试也是失败
- 事务中有外部调用，无法回滚，事务实际无法保障。需要使用two-phase commit，核心是外部调用也能支持rollback

# Weak Isolation Levels

isolation有较大的performance损失，很多DB选择放弃一部分isolation，换取performance

weak isolation会造成bug不只是停留在理论上。现实中发生过造成重大金钱损失，或者招致金融审查的案例

即使一些流行的relational DB，实际也用的是weak isolation。未实现完全的ACID

所以，了解DB的weak isolation的细节，并在实践中结合场景做选择，是很有必要的。 这对开发者提出了更高的要求，而不是完全依赖DB

## Read Committed

>Read committed is a very popular isolation level. It is the default setting in Oracle 11g, PostgreSQL, SQL Server 2012, MemSQL, and many other databases.

 _read commited_ 是transaction isolation的最基础level。即：
- read到的都是committed的数据(no dirty reads)
- write覆盖的也都是committed的数据(no dirty writes)

### No dirty reads(脏读)

write视角：no dirty reads意味着任意writes只有在其所属的transaction committed或者roll back完成后，write带来的变更才对其他reads visibale  
read视角：read拿到的数据都是committed或者roll back完成的终态数据，不存在脏读


### No dirty writes
和No dirty reads理解类似。

通过加行级别写锁的方式，可以解决dirty writes的问题，但是无法完全避免`race condition`。比如7-1图的counter increment场景，后一个transaction在前一个committed后才write，read的也是committed的数据，做到了read commited，但仍然得到错误的结果。  **这里有个疑问，如果有这个race condition的问题，为啥很多流行数据库的default isolation level是read committed？**
![](/images/Pasted%20image%2020240101190402.png)

### Implementing read committed

dirty write：use row-level write locks, 一个transaction在write的时候，需要先拿到**排他写锁**，直到committed或者roll back后，才释放该lock。按照No dirty write的定义，这里的lock是在write开始时获取，在transaction完成时释放

dirty read
- 使用same row-level lock，read时申请lock，read完成后立即释放lock，有write lock的时候read就阻塞。  但是这种方式在write比较慢的情况下，read performance会受到很大影响
- DB remembers both the old committed value and the new value(just two version), write没有commited的时候read old version；commited后read new version

## Snapshot Isolation and Repeatable Read

read committed导致 _read skew_（aka _nonrepeatable read_） 现象

![](/images/Pasted%20image%2020231106191930.png)

Account1/2都是Alice的账户，一共1000元，她操作从Account2向Account1转账100元。        Alice读两个account是同时发起的（transaction），但是由于同时有两个write，虽然满足read committed，但是刚好Alice看到的两个账户结果加起来不等于100

核心问题是，两个write虽然保证了顺序执行，但是没有成为一整个transaction

这种暂时性的非一致性，在下面两种场景下不可接受：
- 数据库backup执行时间长，要backup很多数据。不能backup里既有new version，也有old version。 backup内，预期是同时保持new version，或者同时保持old version
- OLAP。大数据分析场景，query执行复杂且慢，不能一部分读old version，一部分读到new version

_snapshot isolation_ 是针对这类场景的最常用解法，snapshot可以看作多个committed transaction的集合，其中不存在未committed的数据
> it is supported by PostgreSQL, MySQL with the InnoDB storage engine, Oracle, SQL Server, and others


### Implementing snapshot isolation

snapshot isolation也是使用write locks来避免dirty writes。但是这个lock只block其他write，不阻碍read，同样read也不阻碍write
> a key principle of snapshot isolation is readers never block writers, and writers never block readers.

read committed通过write lock，只实现了单个write的ACID。 snapshot isolation在此基础上，可以在read不阻塞的前提下，解决 _read skew_ 问题

这种技术也被称为 _multiversion concurrency control (MVCC)_

snapshot也可以用来实现read committed。不同之处在于，read committed只包含单个write； 而snapshot isolation用于整个transaction，可能包含一系列读写操作

具体实现上，snapshot isolation会在每行数据上增加专门的字段用来记录transaction id，来区分version

- `created_by` field，记录insert这条数据的transaction id
- `deleted_by` field，记录delete这条数据的transaction id。delete为软删除，保留version。等确认没有任何transaction access这条数据的时候，由单独的GC进程执行硬删除

### Visibility rules for observing a consistent snapshot






