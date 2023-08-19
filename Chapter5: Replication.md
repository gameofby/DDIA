本章笔墨着重于应对replication中的数据变更

# Leaders and Followers

基本的结构如下图。leader写，follower读
![](/images/leader-based-replication.png)

## Synchronous Versus Asynchronous Replication
![](/images/leader-based-synchronous-asynchronous.png)

follower 1 is synchronous
1. leader收到follower 1 的同步反馈
2. leader反馈user  write success
3. 其他clients对write可见

synchronous的优劣
1. 优势：一致性强保证
2. 劣势：可能发生的write阻塞

应对synchronous劣势的方式1:
1个follower synchronous，其他的都asynchronous,如果synchronous follower挂了，其他任一asynchronous follower补上

应对synchronous劣势的方式2:
虽然在durability上有折衷，纯异步还是拥有广泛的应用，尤其是follower特别多、地理分散分布的情况

## Setting Up New Followers

1. take snapshot from leader. 通常都有backup机制，snapshot是现成的
2. Copy the snapshot to the new follower node
3. 定位到snapshot之后的所有changes。如：
   >PostgreSQL calls it the log sequence number, and MySQL calls it the binlog coordinates
4. 追上leader的进度后，就可以和其他follower一样，正常接收新的变更

## Handling Node Outages

### Follower failure: Catch-up recovery
follower挂掉恢复时，根据自己的log，定位最后一个处理的write，然后联系leader开始catch up落后的部分，直到catch up

### Leader failure: Failover

自动故障转移的步骤：
1. 判断leader是否failed: 设定超时时间
2. 选择一个新leader: 通常选择changes同步最新（相比于old leader最接近的）的replica。 所有replica同意新leader选举，被称为consensus problem，会在Chapter9详述
3. client、follower都要重新配置，与new leader进行交互。  如果old leader后来又恢复了，系统机制需保证它变为follower，避免多leader冲突

故障转移很容易出错：
- 如果new leader没有和old leader完全同步，会导致一部分write丢失。一般直接丢弃old leader未同步的write。   会违反client的durability预期。      算是在复杂性、durability之间做了折衷，牺牲了部分的durability
- 丢弃write很危险。例如，Github曾经遇到的事故：同步滞后的follower被选为new leader。由于DB使用自增id为主键，并且该id用于redis标识row。  当new leader从滞后的id开始自增，覆盖了一部分old leader已经使用过的id，导致redis上的一些信息被污染
- 有可能发生两个leader共存的情况（aka split brain），互相冲突。
- timeout时长设定。 设长了，failover会很不及时；设短了，又有可能频繁触发本不应该failover的failover，雪崩

## Implementation of Replication Logs
这一小节讨论leader和follower之间同步使用的具体log设计

### Statement-based replication
最直观简单的一种方法。就是leader的每个wirte statement（比如每一条SQL）打log，然后把statement传输到follower一样执行。缺陷：
1. 非确定性函数，如NOW(), RAND()等，多个follower执行后会有diff
2. 自增列，需要每个replica按照和leader一样的顺序执行。  实现这点，会限制并发事务
3. 有副作用的函数，并且副作用不是确定性的。 如触发器、stored procedures, UDF等

以上缺陷可以case by case解决。如非确定性函数，可以在leader先执行拿到结果，再传输给follower。但是边界情况太多，处理起来比较麻烦

MySQL在5.1之前用的就是statement-based replication。新版本的默认方式，已经更换为row-based replication

### Write-ahead log (WAL) shipping
如Chapter2所述的LSM-Tree和B-Tree，都有类似Write-ahead log（WAL）的log存储。可以直接使用WAL作为leader和follower之间同步的数据

- 劣势：WAL比较底层，到了disk block级别，如何解读WAL依赖存储引擎。 WAL和存储引擎耦合太紧密。如果存储引擎升级版本，难以做到向前向后兼容
- replication协议支持版本兼容的重要性：可以做到zero-downtime升级。  方法是，先升级follower，然后通过一次failover实现leader更换

### Logical(row-based) log replication
核心思路：log存储格式和存储引擎解耦。具体如下：

1. insert: log record包含每一列的value
2. delete: 通过主键定位。没有主键的表，log包含待删除数据的每一列value
3. update: 2 + 1。 insert可以只有那些待update的列的value

事务除了产生多条类似的log record，还会在结尾多一条，用于说明以上log已经commit。

MySQL的binlog就使用的Logical Log

这种log还有一种好处，就是便于和外部交互。比如ETL到数仓，或者允许外部自定义index、cache等

### Trigger-based replication
DB自带的replication能力，纯粹由DB来实现，可以应对大部分情况。  个别情况下，如果需要实现一些自定义的replication，灵活性就不太够。因此，一些DB（比如Oracle GoldenGate）提供了工具，可以支持更灵活的replication。    工具一般是借由DB的trigger能力来实现。  可以在发生change的情况下，触发一段custom code来执行自定义的replication

# Problems with Replication Lag
在异步replication的架构中，虽然非常适用于read-scaling（读多写少，可以通过不断拓展node来实现scalability）的场景。但如果replica过多，或者遇到网络问题，难免follower落后（lag）于leader，consistency得不到保障。落后过大会实际地影响到产品和用户。 本节主要介绍3种典型情况，以及解决手段

## Reading Your Own Writes
适用场景：一个用户write的内容，只能被这个用户本身read。  比如个人主页、自己的评论等等

一致性解决方法：`read-after-write consistency` (aka `read-your-writes consistency`)。具体实现：
- 用户可能修改的（比如自己的主页），都从leader读，其他（比如他人的主页）从follower读。 
- 上一条不适用于“应用的大部分内容都可能被所有用户修改”的情况，这样leader要承担大部分写、读，扛不住。  这种情况下，需要增加一些其他手段。  比如：记录最后一次更新时间，紧接着1分钟内的read都从leader读；1分钟之外的，如果监控发现follower滞后超过1分钟，也从leader读，否则就可以直接从follower读
- 客户端记录最后一次write时间戳。follower上读的时候，要求至少该时间戳之前的update replication都要完成，才能读。 时间戳可以是逻辑时钟（标明先后顺序的，比如log sequence number），或者system clock（对各个replica间的clock同步有强依赖）
- 多地域datacenter的情况下，如果必须从leader读，可能需要先路由到leader所在的datacenter

同用户多设备的情况，方法需要进一步设计：
1. 时间戳的记录，不能在单个client上了。需要集中在一个地方
2. 多地域datacenter的情况，也需要额外手段路由到同一个datacenter。  默认情况下，不同设备使用的网络不同，很有可能请求没打到同一个datacenter

## Monotonic（单调） Reads
情景: `moving backward in time`
用户连续多次read（比如刷新网页），后read相比于先read看到的信息反而变少了（比如第一次看到一条新评论，刷新下又没了）。  这种是因为，两次read被随机到两个不同的replica上，后者比前者的replication滞后更严重。如下图
![](/images/moving-back-in-time.png)

解决办法：`monotonic reads`
含义：通过将同一用户的多次read固定到同一个replica等方式，实现一致性。 这种一致性强于最终一致性，弱于强一致性（因为用户只是不会后退，但不代表可以及时看到leader已经发生的所有更新。比如有其他用户新发了评论，有些用户看到了，有些没看到。 
具体实现：如何把同拥护的read请求固定到同一个replica响应？比如哈希拥护id等方式。  如果对应replica挂了，就不得不换一个replica，还是可能打破一致性

## Consistent Prefix Reads
情景：因果性（先后顺序）被破坏。具体见图
![](/images/consistent-prefix-reads.png)

解法：`consistent prefix reads`。听起来很好实现，wirte保持顺序，read按顺序读。  但实际上，尤其在partitioned(sharded)类型的DB种，无法保证全局写的order。  可以尝试把有因果性的wirte都写入到同一个partition，但实际实现起来，可以无法保证执行效率。   有一些算法可以进一步解决，详见书的p186。

## Solutions for Replication Lag
- 对于只能保证最终一致性的系统，要结合具体情况，判断下replication lag的具体影响。 在异步的基础上制造出同步的效果，是解决最终一致性影响的秘诀
- 通过application code来实现read-after-write、monotonic reads、consistent prefix reads等，是复杂的。  这也是transaction存在的意义，DB解决问题，对开发者透明
- transaction在单机早就实现了。 但是到了分布式系统，很多人抛弃了它，出于性能、可用性等原因。或许有些道理，但过于简单化。  Chapter7 and Chapter9会详细讨论解法

# Multi-Leader Replication

## Use Cases for Multi-Leader Replication

### Multi-datacenter operation

datacenter内使用如前所述的leader-follower replication；datacenter之间，leader互相replication
![](/images/multi-datacenter.png)

优势：
1. Performance: 不用所有write都路由到唯一一个datacenter的leader；local解决write，datacenter间的replication异步完成。观感上性能好很多
2. Tolerance of datacenter outages: 如果leader所在的datacenter挂了，不用做failover。
3. Tolerance of network problems: single-leader，write如果是跨地域的，需要依赖于datacenter之间基于公网的网络连接，容易不稳定。multi-leader，write都是在local的datacenter完成，不依赖于跨长距离的公网

劣势：
多datacenter，同一数据的多地域并发modify，可能产生conflict。  需要多datacenter的leader间replication时先解决conflict。


总的来说，multi-leader和一些DB早有的feature之间，可能产生隐患和问题，比如自增id、trigger等。  因此被视为一个存在危险的新feature，应该尽可能避免使用，除非不得不


### Clients with offline operation
以canlendar场景为例，多设备间需要同步，断网状态下calendar也能要继续使用。类似于一个微缩的multi-datacenter replication, leader是每个device的local小DB

### Collaborative editing
多人在线协作的场景。  相比于上个canlendar的例子，特别需要处理的是conflict。 类似multi-datacenter replication的方式，使得多用户可以同步编辑，不用获取lock，但是需要在异步replication的时候处理conflict

## Handling Write Conflicts

### Synchronous versus asynchronous conflict detection
如果通过synchronous的方式检测冲突，就是single-leader了

### Conflict avoidance
solution
> all writes for a particular record go through the same leader

对于不得不更换datacenter的场景（fail; 用户换了地域，最近的datacenter更换），conflict还是会发生

### Converging（收敛） toward a consistent state

为了datacenter间的最终一致性，需要将冲突收敛。有一下几种方式:
1. 每个write赋予一个unique id(如时间戳、随机数、UUID等)，then `pick the write with the highest ID as the winner, and throw away the other writes`. 可能导致data loss
2. 每个replica指定一个unique id，以其中一个id最大的replica的write为准。 也可能导致data loss
3. > Somehow merge the values together—e.g., order them alphabetically and then concatenate them
4. 把冲突记录到单独的地方，之后再图解决。 比如抛给用户（ confluence的wiki，不就这么搞的）

### Custom conflict resolution logic
- On write: 用户可以写一段 `conflict handler` 代码，在detect到conflict的时候调用来解决冲突
- On read: 
  1. store conflict
  2. return all versions when reading
  3. user or application code resolve conflict
  4. write result back

conflict resolution粒度: 通常在row、document粒度，而不是transaction

Automatic Conflict Resolution: 书中提了一些自动解决冲突的新研究成果

## Multi-Leader Replication Topologies
讲leader之间按照怎样的path（topology）左replication。 有三种方式：
![](/images/multi-leader-replication-toloplogy.png)

- MySQL用的circular topology
- > In circular and star topologies, a write may need to pass through several nodes before it reaches all replicas
- circular和star的topologies，相比于all-to-all，容错性较差，一个node fail就可能导致replication停滞。      虽然可以绕过failed node，但是在很多系统中，这个需要手动配置

很多系统`conflict detection`做的很差。所以，使用multi-leader的分布式系统，需要特别注意其具体的实现、充分测试，才能保证不用错。    因为system在设计实现上本身不完善，一不小心就掉坑里了

# Leaderless Replication
- 有leader架构的核心特点：leader来决定write的执行顺序，follower复刻即可
- leaderless早就有了，只是在relational DB时代被遗忘。  直到amazon厂内的Dynamo重新启用。开源的Riak, Cassandra, and Voldemort也开始借鉴。因此被称为 `Dynamo-style`
- leaderless实现上，client直接发给所有的replica，或者由coordinator转发。 但是coordinator不强制write的顺序

## Writing to the Database When a Node Is Down
leaderless设计中，client将write同时发送给各个replica，其中个别失败的node不阻塞client判定write成功； client读取数据时，也要想各个replica都发送请求，根据返回数据的version number，判断哪个数据时最新的。避免有node失败期间遗漏write导致数据滞后

### Read repair and anti-entropy

# Todo
p159, `触发器`, `stored procedures`