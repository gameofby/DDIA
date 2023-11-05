# Partitioning and Replication

结合使用，但又相对独立。 本章will keep things simple and **ignore replication**

![[Pasted image 20230908150420.png]]

# Partitioning of Key-Value Data

如何决定一条数据分配到哪个partition？

需要最终尽可能均匀分配，避免skew（倾斜）导致hot spot（某个partition里数据特别多，或者read/write load集中于某几个partition的热点数据）。  这里有点像bucket sort。需要以均匀分布为前提，才能发挥出威力

random倒是可以做到write均匀，但是无法进一步read key

## Partitioning by Key Range

![[Pasted image 20230912115835.png]]

key按照连续区间划分。如上图，book shelf就是一种很直观的例子，书本按照name的字典序排列。 partition内部也按照key有序排列（类似LSM-Trees），这样查询range scan就会很方便(比如 `event_day between 0101 and 0201`)

数据均匀分布的重要性：避免hot spot

如何均匀分布：按照数据本身的分布来划分partition的boundaries。 有的通过管理员人工划分，有的DB可以自动划分

缺点：有时候hot spot就是key range集中的（比如日期），读/写的热点访问很难避免

## Partitioning by Hash of Key

key -> hash function -> hash key -> partition

可以缓解hot spot问题

hash function需要能保证
- hash key的均匀分布
- 稳定性：在不同的process等下，hash value要相同。 像Java的Object.hashCode()，就不满足这一点

缺点：hash后无法range scan。举例如下：
> In MongoDB, if you have enabled hash-based sharding mode, any range query has to be sent to all partitions [4]. Range queries on the primary key are not supported by Riak [9], Couchbase [10], or Voldemort.

Cassandra适用了一种`compound primary key`，多列组合主键。前一部分主键用于partition，后一部分支持range scan

## Skewed Workloads and Relieving Hot Spots

极端case：hot spot集中于一个key。比如twitter上的大V

data系统尚无法自动解决这类极端问题，需要application层去解决。比如在write/read的时候，在key前拼一个随机数，这个数字需要在application层面记录下来，保证读得时候可以把多份partitione的结果拼得回来


# Partitioning and Secondary Indexes

>Secondary indexes are the bread and butter of relational databases, and they are common in document databases too.

secondary indexes也是Solr、ES等搜索引擎存在的理由
> secondary indexes are the raison d’.tre of search servers such as Solr and Elasticsearch.

## Partitioning Secondary Indexes by Document
![[Pasted image 20231023164027.png]]

如图，document的secondary indexes，就是在各个partition中的reversed index。由于只关注本partition的数据，也被称为 _local index_

这种情况下，一般都要使用 _scatter/gather_ 的方式查询。  即read请求发送到每个partition，然后汇总结果
> Nevertheless, it is widely used: MongoDB, Riak [15], Cassandra [16], Elasticsearch [17], SolrCloud [18], and VoltDB [19] all use document-partitioned secondary indexes.

## Partitioning Secondary Indexes by Term

also called _term-partitioned_, global index

![[Pasted image 20231023170123.png]]

如图，color这个term（这个词来自search engine，按照term建立倒排索引），按照value分布在不同的partition上
> colors starting with the letters a to r appear in partition 0 and colors starting with s to z appear in partition

优点：read更高效，不用做scatter/gather
缺点：write会变慢和变复杂。需要分布式事务来保障原子性，一般异步执行。因此会比较耗时，不会马上看到效果。案例：
> For example, Amazon DynamoDB states that its global secondary indexes are updated within a fraction of a second in normal circumstances, but may experience longer propagation delays in cases of faults in the infrastructure


有些数据库会让你从local index和global index中选

# Rebalancing Partitions

三种变化引出rebalancing
- requests增加 -> add CPUs
- data inc -> add disks
- node fail -> 接手load

rebalancing期待三种最小要求
- rebalancing后保持均匀分布
- rebalancing过程中不停机
- 只挪动必要的数据

## Strategies for Rebalancing

### How not to do it: hash mod N
如果直接使用hash key -> mod N（node数量） -> node，无法适应node数量变化的情况

### Fixed number of partitions
在hash key和node之间，加一层partition，partition数量多于node。  可以屏蔽node变化对key到partition的影响，rebalance也是partition粒度的

下图展示了add one node时，需要rebalancing的partition，只有一部分，且key到partition的映射关系不变。   变化的仅仅是小部分partition和node之间的映射关系
![[Pasted image 20231023183823.png]]

其他好处：把更多的partition分配到更powerful的node上，充分利用

这种rebalancing方法在Riak [15], Elasticsearch [24], Couchbase [10], and Voldemort [25] 中都有应用

选择一个固定的partition数量，rebalancing是更简单的。  但是这个fixed数量不好选
- 数量过多：partitions的management overhead过高，比如IO、文件句柄等
- 数量过少：无法适配数据和请求的急剧增长，单partition的数据量和请求量过大，rebalancing和recovery成本都过高


### Dynamic partitioning
单个partition设定一个范围阈值

- > upper threshold：1 partition split to 2 partition。split后的partition也可能需要transfer到其他node，以保证node之间的load 屏很
- < lower threshold: 2 partition merge to 1 partition。

比较像B-Tree的变化方式

为了避免数据库从empty开始只有一个partition，可以预先设定一个range分割。 前提是需要知道(hash) key range的分布

### Partitioning proportionally to nodes
一种在Cassandra等DB中使用的方法，固定单个node上partition个数，随着数据的增长，增加node。单个partition上的数据量会趋于稳定

新增node时，会从已有partitions中随机选取一些，half split，分一半给new node


## Operations: Automatic or Manual Rebalancing
一般是两者结合。全自动的风险较高，容易和 _automatic failure detection_ 一起引起级联失败

# Request Routing
client如何知道某个request最终请求哪个node上的哪个partition？
- 连接任意node，由node来判断实际partition所在的node。  第一个node充当router作用
- 增加一个中间routing层，专门来做这个事情
- client本身知道应该请求哪个node（维护了routing的信息）
![[Pasted image 20231024152841.png]]

很多DB（LinkedIn’s Espresso, HBase, SolrCloud, and Kafka等）内置了ZK，维护partitions -> nodes的信息
![[Pasted image 20231024152918.png]]

Cassandra and Riak采用了方法2，实现了一种称为 _gossip protocol_ 的协议，用来在nodes之间传播routing metadata，相当于每个node都维护了一份metadata。   新增了复杂心，但是规避了依赖一个额外的ZK服务

## Parallel Query Execution
实际应用中，query都比较复杂，query optimizer会把原始query拆解为多个自任务，其中很多任务可以并行执行，以提升大数据查询的效率。 会在Chapter 10详细介绍