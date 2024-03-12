# Data structures that power your database
## Hash Indexes

**基本结构(p72-p75)：**

使用hashmap。hashmap的value存储实际(key, value)在disk file中的offset

适用的场景？
>Bitcask (the default storage engine in Riak). 
>well suited to situations where the value for each key is updated frequently

![[Pasted image 20230827201508.png]]

![[Pasted image 20230827195927.png]]

如何避免append file超过了disk space?
使用segment + compaction
- segment：将log分割为certain size的segment，如果append已经达到了size，就close，创建一个新的segment来append
- compaction：起一个background thread，对历史的segments做merge，只保留most recent的数据（对于key value，保留最新的value； 如果是复杂的store，需要按照既定的规则计算，比如doris，每个aggregate column都有定义对应的aggregate function）

详细步骤：
1. read write请求，在**可用**的最新（最近compaction完的版本）的segments上操作。同时起background thread做compaction
2. compaction不是in-place，而是开辟新的区域输出结果
3. compaction结束后，`switch read requests to using the new merged segment`，然后old segments可以简单地删掉
4. 内存中，每个segment有对应的hashmap
5. read write请求，会先在最新的hashmap中找，找不到就换次新的，以此类推
6. 因为有compaction，hashmap(segment)的数量不会太多


其他实现细节
   - File format: binary format. bytes length of string + raw string
   - delete: add a special record called tombstone, for deleting signal in compaction, 惰性删除
   - crash recovery
	   - slow: read all **segments** on disk to restore hashmap in memory
	   - fast: store **snapshot of hashmap**. load it to memory when recovering
   - partially written records: use checksum to find it
   - concurrency control: only one writer thread running

**为什么使用append-only而不是update的方式存储？(p75)**

1. 顺序写入比随机写快，尤其在HDD上。  即使SSD顺序写也更快，原因见：page83
2. crash recovery和concurrency更简单
3. compaction避免了文件碎片化

**hash table index的局限(p75)：**

1. 依赖内存大小。  如果key数量极大，就不太适用。  基于硬盘实现hashmap，非常复杂，成本较高
2. range query查询低效



## SSTables（Sorted string table） and LSM-Trees

### 基本结构(p76)
在hash index的segment基础上，要求segment内按照key排序。以及，merged segment中，一个key只能出现一次（compaction保证了这一点）

### 优势（p76-p77）
1. compaction: 归并排序即可，即使待merge的segments超过了memory size也不影响。同key但value不同的，丢弃更早的value
2. 内存索引: 因为segment中key是有序的，内存中不必再保留所有的key index。使用稀疏索引，找到被检索key的segment offset的范围，再进一步在范围内的segment搜索即可
3. 压缩：稀疏索引对应的block，可以压缩存储，节省磁盘I/O和space
4. 支持高效range query，由于disk write是顺序的，可支持很高的写吞吐

### 如何构建sorted index？
1. 数据结构：内存中维护一个平衡树（如红黑树, AVL tree），有序。 一般叫做**memtable**
2. spill：红黑树到一定阈值，`typically a few megabytes`，spill到磁盘SSTable segment。这里和上一节的hash index不太一样，index和SSTable是分离的
3. read：优先查内存。查不到的查磁盘segment，由近及早
4. compaction：定期后台合并，丢弃较早的value、删除的key

### 劣势
crash recovery，内存直接丢了，因为内存这部分是尚未spill的segment，disk中的segment没有这部分数据

解法：write-ahead， write之前，先打印log（类似binlog），crash后从log恢复。  如果log对应的write已经落盘为SSTable，对应的log就可以删除了

### LSM-Tree历史沿革(p78-p79):
上述算法，Patrick O’Neil等人先使用Log-Structured Merge-Tree（LSM-Tree）来描述；

**SSTable**和**memtable** 的名字来自于Google Bigtable的论文；

LevelDB，RocksDB使用的KV引擎受到到Bigtable启发；

ElasticSearch 和Solr 中的Lucene引擎，也使用了类似的idea，来构建key(item) -> value(document id list) index（倒排索引）

### 性能优化
bloom filter: 可以快速告诉你，一个元素ju
   >It can tell you if a key does not appear in the database

compaction的顺序和时机
   1. size-tiered（HBase）： "新的、小的"segment合入到"旧的、大的"
   2. leveled（LevelDB and RocksDB）：（寥寥几句话，没太看懂...） TODO：可以详细研究
   Cassandra两种都支持



## B-Trees

### structure(p80-p81)
![](/images/b-tree_structure.png)

block/page: 对应图中一个树的每一个节点。size是固定的，类似disk底层的管理方式。  不像log-structure index, 是可变大小。 reference代表了子树的数据范围。每个page中可容纳的reference数量，叫做 _branching factor_，一般大小是几百

存储在disk上，而不是内存

update: 从root开始，顺着树查询

insert: 查询，然后在合适的位置插入。  如果当前page超出 _branching factor_，需要分裂为两个page，更新父亲节点。  有可能递归向上传播。 这样可以保证树的平衡，可以在O(logn)查询效率

level：一般3、4层就够
   >A four-level tree of 4 KB pages with a branching factor of 500 can store up to 256 TB
   


### Making B-trees reliable(p82)
1. 现象：B-tree的update操作是直接disk原地更改，包括insert时候可能触发的page split
2. crash recovery: a write-ahead log（WAL），就是在实际执行write操作前，先把操作写到append-only日志中，以待crash的时候用来恢复
3. concurrency control: latches (一种轻量级锁)

### Optimization (p82-p83)
1. copy-on-write: write page的时候copy page到一个新的location再write，parent page用版本管理，新版本reference指向新的location。 版本管理也可以用来做concurrent control，详见P237
2. abbreviating key: 缩写key，仅保证range scan的功能即可。这样可以增加page的branch factor
3. B+tree
>pointer to siblings page from left to right in leaf level, which allows scanning keys in order without jumping back to parent pages


## Compare B-Trees and LSM-Trees

### LSM-tree的优势
1. lower write amplification: 因此而拥有更好的写吞吐能力
   >the more that a storage engine writes to disk, the fewer writes per second it can handle within the available disk bandwidth.
2. reduced fragmentation: 顺序写入，不像B-tree那么碎片化；更容易压缩；

### LSM-tree的劣势

compaction额外占用资源，可能影响到正在执行的读写操作
>compaction process can sometimes interfere with the performance of ongoing reads and writes.

LSM-tree中，同一个key可能存在于多个segment中；B-tree相反，都在一个page里，更好做读写隔离（如事务的隔离）


## Other Indexing Structures

### Storing values within the index？

`heap file`: 实际存储value的地方。index中，value存的是heap file的ref。 这种一般叫`nonclustered index`

`clustered index` 
>storing all row data within the index

比如inno db，primary index和row data是存在一起的，secondary index指向primary index

good
>speed up reads

bad
1. 需要处理index和数据、多个index之间的一致性问题；
2. duplicate导致额外的存储消耗；
3. write需要写多个地方



### Multi-column indexes（多列索引）
1. `concatenated index` 就是把多个key拼起来而已（类似doris的前缀索引），只能按照拼的顺序做前缀筛选，不能单独筛选非一级索引
2. `Multi-dimensional index`：支持多列**同时**筛选，而不是各自筛选再求交集（B-tree等可以这么实现）。 一般可以用R-tree实现。实际应用：
> PostGIS implements geospatial indexes as R-trees using PostgreSQL’s Generalized Search Tree indexing facility

### Full-text search and fuzzy indexes
常用于搜索引擎，支持纠错（一定的编辑距离）、同义词等，属于信息检索的范畴。  一般用到 `trie` 等数据结构



### Keeping everything in memory
完全使用内存，然后使用snapshot，分布式，带电池的RAM，NVM（断电后仍可保存数据的内存新型硬件）等方式提供reliability

反直觉地，in-memory database快并不是因为不用从disk读。  一般内存足够大的话，OS也会把最近用的cache到memory。 主要的“快”来自于不用把内存里的数据合适方式编码后存储到disk。  因此，in-memory还有一个优点，是可以提供更复杂的数据结构（如redis）

另外，即便使用类似OS的方式，把很久没用到的spill到disk，再把最近用到的加载到内存。  但是相比OS是以page的粒度，in-memory db可以以record的粒度来做，粒度更精细


# Transaction Processing or Analytics?
这节没讲的内容都太熟了，没啥可以记的，就放个比较olap和oltp的图吧
![](/images/diff-olap-oltp.png)

# Column-Oriented Storage

1. 列存：想法很简单，就是把每一列的相近数据存在一起，列间拆分。而不是把相近的行存在一起。
2. 试用场景：analytics/olap, 查询中scan的列较少（4～5，很少select \*），但是scan大量的row。 如果行存，会导致要scan几乎所有的数据进内存

![](/images/relational2column.png)

如图，列存中有一个关键点，每列中包含的行顺序要一致

### Column Compression

**bitmap encoding** 是一种有效的压缩方法

1. column的distinct value肯定少于row的disitnct数
2. 用n个不同的数代表column所有的distinct value，每个distinct value对应一个独立的bitmap。  bitmap中的每个bit位对应一个row，row的该列value如果和bitmap代表的value相同，bitmap对应位置为1，否则为0
3. 当数据量很大的时候，bitmap中的1会很稀疏。可以使用run-length encoding解决。如下图中的后半部分
4. 用bitmap之间位运算来实现OR、AND where conditions。得益于每个bitmap的二进制位数相同（行数相同）；且即使column不同，行的顺序也相同
```sql
   WHERE product_sk IN (30, 68, 69)
   -- Load the three bitmaps for product_sk = 30, 68, 69
   -- calculate the bitwise OR of the three bitmaps
   -- then，那些bit位为1的，就是要查询的行。可以借此拿到行对应的索引（顺序相同）

   WHERE product_sk = 31 AND store_sk = 3
   -- Load the bitmaps for product_sk = 31, store_sk = 3
   -- calculate the bitwise AND
   -- 符合这个条件的行，行内product_sk=31和store_sk=3同时成立，即两个bitmap的同一bit位（对应行），值都是1
```

详细见图：
![](/images/column-compression.png)

**memory bandwidth and vectorized processing**
这块书中描述比较简略，可以参考书中给的引用资料深入

### Sort Order in Column Storage
column如果要重排序（即在原本append插入顺序的基础上，重排序），需要同row的其他column一起排，否则将找不到属于同row的其他column

DBA可以参照日常query分布来决定按照哪个column来排序(如date)。也可以分一级、二级sort key

优点
1. 加速查询（前缀索引的效果）；
2. 排序的列更容易压缩。因为相同的值连续排列

多种排序共存：可以在分布式replication存储上，使用不同的排序方式。然后在查询的时候，挑选最适合当前query的排序。 和row-oriented不同的地方在于，column-orinted实实在在存了多份原始数据（replication）。而row-oriented可以只存不同的索引，原始数据只有一份

### Writing to Column-Oriented Storage

基于以上对column-oriented的描述，虽然对read很有利，但是write就复杂了

1. 类似LSM-tree，先在内存里积累，内存里是一个sorted的数据结构
2. 然后和disk上的数据merge，保持数据在disk是列存的、有序的
3. 查询需要结合memory和disk上的数据

>This is essentially what Vertica does

### Aggregation: Data Cubes and Materialized Views
物化视图：轻度聚合，单独存储。 不像OLTP db里的视图，只是一层虚拟视图，查询视图的sql会转为查询table的sql，做不到加速查询，只是抽象对外的接口

Doris就有这个feature

1. 优势：加速SUM等特定的聚合查询
2. 劣势：写成本较大（但是对于“重读”的数仓，用空间来换时间还是划算的）；对应需要从最细粒度原始数据的查询无能为力，比如查询分位值等

