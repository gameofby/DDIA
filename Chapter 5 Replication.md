本章笔墨着重于阐述replication中发生数据变更后，consistency如何保持

# Leaders and Followers

基本的结构如下图。leader写，follower读
![](/images/leader-based-replication.png)

## Synchronous Versus Asynchronous Replication
![](/images/leader-based-synchronous-asynchronous.png)

follower 1 is synchronous
1. leader收到follower 1 的同步反馈
2. leader反馈user write success
3. 其他clients对write可见

synchronous的优劣
- pros(consistency)：一致性强保证。要么都成功，要么都失败
- cons(availability)：可能由于某一个环节异常，发生的write阻塞

应对synchronous劣势的方式
1. 1个follower synchronous，其他的都asynchronous。如果synchronous follower挂了，其他任一asynchronous follower补上。like quorum protocol
2. 都asynchronous：虽然在consistency上有折衷，纯异步还是拥有广泛的应用，尤其是follower特别多、地理分散分布的情况

## Setting Up New Followers

1. take snapshot from leader. 通常都有backup机制，snapshot是现成的
2. copy the snapshot to the new follower node
3. 定位到snapshot之后的所有changes。如：
   >PostgreSQL calls it the log sequence number, and MySQL calls it the binlog coordinates
4. 追上leader的进度后，就可以和其他follower一样，正常接收新的变更

## Handling Node Outages

### Follower failure: Catch-up recovery
follower挂掉恢复时，根据自己的log，定位最后一个已经处理的write，以此为起点。联系leader开始catch up落后的部分，直到catch up

### Leader failure: Failover

自动故障转移的步骤：
1. 判断leader是否failed: 设定超时时间
2. 选择一个新leader: 通常选择changes同步最新（相比于old leader最接近的）的replica。 所有replica同意新leader选举，被称为consensus problem，会在Chapter9详述
3. client、follower都要重新配置，与new leader进行交互。  如果old leader后来又恢复了，系统机制需保证它变为follower，避免多leader冲突

故障转移很容易出错：
- 如果new leader没有和old leader完全同步，会导致一部分write丢失。一般直接丢弃old leader未同步的write。   会违反client的durability预期。      算是在复杂性、durability之间做了折衷，牺牲了部分的durability
- 丢弃write很危险。例如，Github曾经遇到的事故：**数据同步滞后于leader的follower**被选为new leader。由于DB使用自增id为主键，并且该id用于redis标识row。  当new leader从滞后的id开始自增，覆盖了一部分old leader已经使用过的id，导致redis上的一些信息被污染
- 有可能发生两个leader共存的情况（aka split brain），互相冲突。
- timeout时长设定很讲究。 设长了，failover会很不及时；设短了，又有可能频繁触发本不应该failover的failover，雪崩

## Implementation of Replication Logs
这一小节讨论leader和follower之间同步使用的具体log设计

### Statement-based replication
最直观简单的一种方法。就是leader的每个wirte statement（比如每一条SQL）打log，然后把statement传输到follower一样执行。缺陷：
- 非确定性函数，如NOW(), RAND()等，多个follower执行后会有diff
- 自增列，需要每个replica按照和leader一样的顺序执行。  实现这点，会限制并发事务
- 有副作用的函数，并且副作用不是确定性的。 如触发器、stored procedures, UDF等

以上缺陷可以case by case解决。如非确定性函数，可以在leader先执行拿到结果，再传输给follower。但是边界情况太多，处理起来比较麻烦

MySQL在5.1之前用的就是statement-based replication。新版本的默认方式，已经更换为row-based replication

### Write-ahead log (WAL) shipping
如Chapter2所述的LSM-Tree和B-Tree，都有类似Write-ahead log（WAL）的log存储。可以直接使用WAL作为leader和follower之间同步的数据

劣势：WAL比较底层，到了disk block级别，如何解读WAL依赖存储引擎。 WAL和存储引擎耦合太紧密。如果存储引擎升级版本，难以做到向前向后兼容

**replication协议**支持版本兼容的重要性：可以做到zero-downtime（不停机）升级。  方法是，先升级follower，然后通过一次failover实现leader更换

### Logical(row-based) log replication
核心思路：log存储格式和存储引擎解耦。具体如下：

1. insert: log record包含每一列的value
2. delete: 通过主键定位。没有主键的表，log包含待删除数据的每一列value
3. update: 2 + 1的事务。 insert可以只有那些待update的列的value

事务除了产生多条类似的log record，还会在结尾多一条，用于说明以上log已经commit。

MySQL的binlog就使用的Logical Log

这种log还有一种好处，就是便于和外部交互。比如ETL到数仓，或者允许外部自定义index、cache等

### Trigger-based replication
DB自带的replication能力，纯粹由DB来实现，可以应对大部分情况。  个别情况下，如果需要实现一些自定义的replication，灵活性就不太够。

因此，一些DB（比如Oracle, GoldenGate）提供了工具，可以支持更灵活的replication。    工具一般是借由**DB的trigger能力**来实现。  可以在发生change的情况下，触发一段custom code来执行自定义的replication

# Problems with Replication Lag
在异步replication的架构中，虽然非常适用于read-scaling（读多写少，可以通过不断拓展node来实现scalability）的场景。但如果replica过多，或者遇到网络问题，难免follower lag（落后）于leader，consistency得不到保障。lag过大会实际地影响到产品和用户。 本节主要介绍3种典型lag情况，以及解决手段

## Reading Your Own Writes
场景：一个用户write的内容，只能被这个用户本身read。  比如个人主页、自己的评论等等。 如下图，user发了一条新评论，但是如果是紧接着从follower 2 read，由于leader和follower2的同步延迟，user读不到他刚刚发的评论

![[Pasted image 20230901161047.png]]

因此，这种场景下需要`read-after-write consistency` (aka `read-your-writes consistency`)。即只保障user write的变更，在刷新页面等操作后，可以马上看到（read）

可能的解法如下：
- 用户可能修改的（比如自己的主页），都从leader读，其他（比如他人的主页）从follower读。 
- 上一条不适用于“应用的大部分内容都可能被所有用户修改”的情况，这样leader要承担大部分写、读，扛不住。  这种情况下，需要增加一些其他手段。  比如：记录最后一次更新时间，紧接着1分钟内的read都从leader读；1分钟之外的，如果监控发现follower滞后超过1分钟，也从leader读，否则就可以直接从follower读
- 客户端记录最后一次write时间戳。follower上读的时候，要求至少该时间戳之前的update replication都要完成，才能读。 时间戳可以是逻辑时钟（标明先后顺序的，比如log sequence number），或者system clock（对各个replica间的clock同步有强依赖）。    同用户多设备的情况，方法需要进一步设计：
	- 时间戳的记录，不能在单个client上了。需要集中在一个地方
	- 多地域datacenter的情况，也需要额外手段路由到同一个datacenter。  默认情况下，不同设备使用的网络不同，很有可能请求没打到同一个datacenter
- 多地域datacenter的情况下，如果必须从leader读，可能需要先路由到leader所在的datacenter



## Monotonic（单调） Reads
场景：user多次read，请求打到了不同的asynchronous followers，有可能后read相比于先read看到的信息反而变少了（比如第一次看到一条新评论，刷新下又没了）。因为每个follower异步 catch up leader数据的进度不同

> moving backward in time

![](/images/moving-back-in-time.png)

解决办法：`monotonic reads`

含义：通过将同一用户的多次read固定到同一个replica等方式，实现一致性。 这种一致性强于**最终一致性**，弱于**强一致性**（因为用户只是不会后退，但不代表可以及时看到leader已经发生的所有更新。比如有其他用户新发了评论，有些用户看到了，有些没看到） 

具体实现：如何把同user的read请求固定到同一个replica响应？比如哈希user id等方式。  如果对应replica挂了，就不得不换一个replica，还是可能打破`monotonic reads`

## Consistent Prefix Reads
场景：因果性（先后顺序）被破坏。比如聊天的场景，同步人看到的对话顺序不同，破坏了上下文的前后依赖、因果性。具体见图：
![](/images/consistent-prefix-reads.png)

解法：`consistent prefix reads`

听起来很好实现，wirte保持顺序，read按顺序读。  但实际上，尤其在partitioned(sharded)类型的DB种，无法保证**全局写的order**。  可以尝试把有因果性的wirte都写入到同一个partition，但实际实现起来，无法保证执行效率。   有一些算法可以进一步解决该问题，详见p186。

## Solutions for Replication Lag
抽象总结一下：

对于只能保证最终一致性的系统，要结合具体情况，判断下replication lag的具体影响。 在异步的基础上制造出同步的效果，是解决最终一致性影响的秘诀

通过application code来实现read-after-write、monotonic reads、consistent prefix reads等，是复杂的。  这也是transaction存在的意义，DB解决问题，对开发者透明

transaction在单机早就实现了。 但是到了分布式系统，很多人抛弃了它，出于性能、可用性等原因。或许有些道理，但过于简单化。  Chapter7 and Chapter9会详细讨论解法

# Multi-Leader Replication

普通的分布式架构，是一主多从。 这里讨论第一种特殊情况：**多主**

## Use Cases for Multi-Leader Replication

### Multi-datacenter operation

datacenter内使用如前所述的leader-follower replication；datacenter之间，leader互相replication
![](/images/multi-datacenter.png)

优势：
- Performance: 不用所有write都路由到唯一一个datacenter的leader；local解决write，datacenter间的replication异步完成。观感上性能好很多
- Tolerance of datacenter outages: 如果leader所在的datacenter挂了，对应的datacenter等着failover就行。 其他datacenter不受影响
- Tolerance of network problems: single-leader，write如果是跨地域的，需要依赖于datacenter之间基于公网的网络连接，容易不稳定。multi-leader，write都是在local的datacenter完成，不依赖于跨长距离的公网。  只有datacenter leader间的数据同步，需要经过公网

劣势：
多datacenter，同一数据的多地域并发modify，可能产生conflict。  需要多datacenter的leader间replication时先解决conflict。


总的来说，multi-leader和一些DB早有的feature之间，可能产生隐患和问题，比如自增id、trigger等。  因此被视为一个存在危险的新feature，应该尽可能避免使用，除非不得不用


### Clients with offline operation
以calendar场景为例，多设备间需要同步，断网状态下calendar也能要继续使用。类似于一个微缩的multi-datacenter replication（每个client相当于一个datacenter） , leader是每个device的local小DB

### Collaborative editing
多人在线协作的场景。  相比于上个canlendar（每个人管理自己的，不太涉及多人）的例子，特别需要处理的是conflict。 类似multi-datacenter replication的方式，使得多用户可以同步编辑，不用获取lock，但是需要在异步replication的时候处理conflict

## Handling Write Conflicts

multi-leader架构下，handle write conflicts是最大的问题

### Synchronous versus asynchronous conflict detection
如果通过synchronous的方式（leaders间写同步，先完成replication同时检测conflict，再返回给用户结果。  如果有conflict，直接抛出来）检测冲突，本质上single-leader一样了。    丢失了asynchronous架构的好处

### Conflict avoidance
muilti-datacenter的场景下，处理conflicts的方法其实很poorly。直接从源头避免conflict产生，或许是个最简单的办法

特定record对应的write，都强制走同一个datacenter的leader。常常，一个用户的请求一般都会打到同一个datacenter，一般是物理距离用户最近的datacenter
> all writes for a particular record go through the same leader

对于不得不更换datacenter的场景（datacenter fail, 用户搬家或者出差），conflict还是会发生

### Converging（收敛） toward a consistent state

为了datacenter间的最终一致性，需要将冲突收敛。有一下几种方式:
- 每个write赋予一个unique id(如时间戳、随机数、UUID等)，then `pick the write with the highest ID as the winner, and throw away the other writes`. 可能导致data loss
- 每个replica指定一个unique id，以其中一个id最大的replica的write为准。 也可能导致data loss
- `Somehow merge the values together—e.g., order them alphabetically and then concatenate them`
- 把冲突记录到单独的地方，之后再图解决。 比如抛给用户（ confluence的wiki，不就这么搞的）

### Custom conflict resolution logic
- On write: 用户可以写一段 `conflict handler` 代码，在detect到conflict的时候调用来解决冲突
- On read: 
  1. store conflict
  2. return all versions when reading
  3. user or application code resolve conflict
  4. write result back

conflict resolution粒度: 通常在row、document粒度，而不是transaction

Automatic Conflict Resolution: 书中提了一些自动解决冲突的新研究成果，见p174

## Multi-Leader Replication Topologies
讲leader之间按照怎样的path（topology）进行replication。 有三种方式：
![](/images/multi-leader-replication-toloplogy.png)

MySQL默认只支持circular topology

node不只要接受和发送write，还要作为中间节点做转发
> In circular and star topologies, a write may need to pass through several nodes before it reaches all replicas

circular和star的topologies，相比于all-to-all，容错性较差，一个node fail就可能导致replication停滞。      虽然可以绕过failed node，但是在很多系统中，这个需要手动配置

很多系统`conflict detection`做的很差。所以，使用multi-leader的分布式系统，需要特别注意其具体的实现、充分测试，才能保证不用错。    因为system在设计实现上本身不完善，一不小心就掉坑里了

# Leaderless Replication
普通的分布式架构，是一主多从。 这里讨论第一种特殊情况：**无主**

有leader架构的核心特点：leader来决定write的执行顺序，follower复刻即可

leaderless早就有了，只是在relational DB时代被遗忘。  直到amazon厂内的DynamoDB重新启用。开源的Riak, Cassandra, and Voldemort也开始借鉴。因此被称为 `Dynamo-style`

leaderless实现上，client直接发给所有的replica，或者由coordinator转发。 但是coordinator不强制write的顺序

## Writing to the Database When a Node Is Down
leaderless设计中，client将write同时发送给各个replica，其中个别失败的node不阻塞client判定write成功； 

client读取数据时，也要向各个replica都发送请求。根据返回数据的version number，判断哪个数据是最新的。避免有node失败期间遗漏write，导致client读到滞后的数据。详细见: “Detecting Concurrent Writes” on page 184

![[Pasted image 20230902194231.png]]

### Read repair and anti-entropy
replication是要保证各个replica的最终一致性的。上小节的offline node最终回归后，如何catch up遗漏的数据呢？

Dynamo-style类DB常用的解法有两种：
- `read repair` client把读到的up-to-date 的value，回写到那些滞后的replica中。 对应上节的例子，就是把version=7的value回写到version=6的replica 3中。   这种对于application一直不read的value，就无法catch up，丢失了部分durability
- `anti-entrophy process` background起一个进程，专门来检测和copy那些replica遗漏的数据

### Quorums for reading and writing
quorum是“法定人数”的意思，更详细的解释，是维持组织运行需要的最少人数

```
n 总replica数
w 最少的write成功replica数
r 最少的read请求的replica数
```

只要w + r > n，read就一定能读到up to date的value

1. 第1种理解：等价于r > n - w。n - w是最多可能write失败的replica数，read的时候，至少得read到一个up to date，所以r得大于n - w
2. 第2种理解：w和r之间存在overlap，read至少能读到一个up to date的replica

`r, w` ，就可以被称作quorum，可以看作write和read的时候需要的最少投票数。  一般会作为replica的配置超参数

`n, r, w` 具体怎么配呢？
- 一般情况
>A common choice is to make n an odd number (typically 3 or 5) and to set `w = r = (n + 1) / 2 (rounded up)`

- 写少读多的情况
w = n, r = 1. 这样读起来会很快，因为只用等一个replica成功返回。 但是，只要有一个replica write出问题，集群就会进入无法write的状态


实际上，读写请求是发送到所有n个replica的，w和r只影响client等待几个成功ack。  这样更合理，因为n个replica可能有快有慢


## Limitations of Quorum Consistency

即使实现了`w + r > n` ，仍然可能存在一些edges，可能return stale values

- sloppy quorum？ 等183
- 写并发：安全的solution是merge concurrent writes。 但是如果merge是基于timestamp，最近的write wins，有可能由于clock skew(时钟偏差)导致真正latest 的write被丢弃
- 读写并发：write小于w，wirte阻塞，但是未回滚已经成功的部分replica。这时候read仍然可进行，old new的value都有可能被读到
- fail over：如果一个持有new value的replica fail over后，是从old value replica同步的数据。fail over后，实际持有new value的replica可能小于w。  打破了quorum condition
- 即使以上提到的情况都ok，也仍有可能有其他edge cases。
> we shall see in “Linearizability and quorums” on page 334

实践中，没有quorum condition那么简单，Dynamo-style DB只能保证最终一致性

### Monitoring staleness

leader-based DB: 监控比较简单，write在各个node都是同序的，比价下log number，就知道follwer有没有lag了

leaderless DB: 很难。而且如果只使用了read repair来避免stale，staleness可能很严重，尤其是不怎么会被read的value

reference \[48\] 有一些相关研究，但是仍不是常见的做法

## Sloppy Quorums and Hinted Handoff

核心点是，集群真正的nodes数量大于n。 这样，当实际执行n、w、r的n个node的quorum被打破的时候，wirte仍可以在n之外的其他node上执行。  待n内的节点恢复后，再把数据移交回（hinted handoff）n个node

wirte不因为quorum打破而停滞，提升了集群的durability

书中举了个很生活化、很好理解的例子：
> By analogy, if you lock yourself out of your house, you may knock on the neighbor’s door and ask whether you may stay on their couch temporarily. Once the network interruption
> Once you find the keys to your house again, your neighbor politely asks you to get off their couch and go home.


各家DB的应用
> Sloppy quorums are optional in all common Dynamo implementations. In Riak they are enabled by default, and in Cassandra and Voldemort they are disabled by default

### Multi-datacenter operation

**Cassandra and Voldemort**
multi-datacenter总共有n个nodes，支持配置每个datacenter的node数量。 client会将writes发往所有datacenter，但是同步只等待local datacenter的返回达到quorum。 发往非local的writes，是异步的

**Riak**
write只发往local datacenter，n也只计算local。  multi-datacenter中间，使用前文提到过的multi-leader replication进行数据同步

## Detecting Concurrent Writes

leaderless replication，多个client可能并发写同一个key。由于网络延迟等原因，并发write达到不同node的顺序可能不同，甚至丢失一些write。  最终导致不同node的value不一致。 如下图：

![[Pasted image 20230906174735.png]]

### Last write wins (LWW, discarding concurrent writes)

实际上没有先后，因为多个client的request是并发的

可以人为强制指定“order”，比如按照write的timestamp，只保留最新的write。 这种方式叫做last write wins (LWW)

> LWW is the only supported conflict resolution method in Cassandra [53], and an optional feature in Riak

LWW的缺陷
- 非最新的write会被discard，还是丢弃了数据的。 如果timestamp因为clock不一致而错误，丢弃的数据就找不回来了
- LWW也有可能discard非并发的数据，会在“Timestamps for ordering events” on page 291 讨论

### The “happens-before” relationship and concurrency

A happens-before B
- B knows about A
- B depends on A
- B builds upon A in some way

concurrent != 时间上同时发生。而是是否互相满足happens-before
>two operations are concurrent if neither happens before the other


### Capturing the happens-before relationship

介绍一种可以capture concurrent operations的算法

场景：多个client同时往一个cart添加食品。（符合到店多个人扫同一个二维码点餐）

![[Pasted image 20230908121514.png]]

server可根据version number判断是否存在concurrent writes

算法works as follows
- server为每个key维护一个version number。每次write，server version number <= write version number的数据，被覆盖；其他的被保留。返回数据的version number++
- A client must read a key before writing。并且，write和上一次read到的数据，要进行merge。再发送write request

这样下来，write就不会丢（silently dropped）

### Merging concurrently written values

如上节，client需要merge并发之下的多次写结果。如果只是cart这个例子，merge很简单。但是如果允许用户delete cart中的item，merge就很麻烦了

有delete操作的情况下，并发merge可以采用类似之前在hash index中提到的，tombstone的方式。打标记，软删除

Riak还创造了专门的数据结构(called CRDTs)，用来做 _Automatic Conflict Resolution_ 

### Version vectors

上文的cart例子，是只有一个replica。如果有多个replica呢？

每个replica上的每个key，对应唯一的version number，单独increment

> The collection of version numbers from all the replicas is called a version vector













