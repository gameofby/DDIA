# The Slippery Concept of a Transaction

## The Meaning of ACID

### Atomicity
很熟悉了，无须过多解释
### Consistency
这个词有点被滥用了，在很多场景意思都不同，书中举例了四种场景

在ACID中，consistency含义是，在应用层面有一些不变量，一定要保持其状态不变

因为是应用层面的概念，所以需要在应用代码逻辑中保证。   如果应用里写入了打破invariant的数据 ，DB也无能为力
> Thus, the letter C doesn’t really belong in ACID.
### Isolation
并发场景（race condition）的概念。

经典教材也会把isolation称为 _serializability_，即每个transaction都可以假定他们是唯一正在运行的transaction，不会被其他transaction影响

实际应用中，极少有DB实现isolation，因为对DB性能影响很大

### Durability
含义：不丢数据

- single node DB：保存到了disk or SSD（相对稳定的存储介质）。以及，存有write ahead log
- replicated DB：`data has been successfully copied to some number of nodes`。transaction需要在replication完成后才commit

不存在完美的durability。 比如极端情况下，所有的存储都挂了，切不可恢复

## Single-Object and Multi-Object Operations

### Single-obejct writes
single object的操作也是需要atomicity和isolation的，否则会遇到以下情况
- 较大的object write了一半网络断了
- overwrite遇到了断电，覆盖一半？还是保留old object
- 并发write read，write没执行完的时候，read是否能读到中间态的write新数据？

### The need for multi-object transactions

Do we need multi-object transactions at all? 只有single object的transaction保障够吗？或许一些简单的场景是ok的，但是以下例子表明了multi-obejct transaction的必要性

- 关系型数据库里的外键
- 图数据库里的点 - 边 - 点
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

weak isolation造成bug不只是停留在理论上，实际中发生过造成重大金钱损失，或者招致金融审查

即使一些流行的relational DB，实际也用的是weak isolation。未实现完全的ACID

所以，了解DB的weak isolation的细节，并在实践中结合场景选择，会更有必要。 这对开发者提出了更高的要求，而不只是完全依赖于DB

## Read Committed

transaction isolation的最基础的level，就是 _read commited_ 。即：
- read到的都是committed的数据
- write覆盖的也都是committed的数据
### No dirty reads

- not committed data: 看到了transaction的部分数据
- not roll-back data：看到了未回滚完成的数据

### No dirty writes





