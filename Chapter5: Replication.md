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
2. 选择一个新leader: 通常选择changes同步最新（相比于old leader最接近的）的replica。 所有节点同意新leader选举，被称为consensus problem，会在Chapter9详述
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



# Todo
p159, `触发器`, `stored procedures`