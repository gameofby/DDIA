向前兼容：新代码可以读老代码生成的老数据。或者对于service间调用，是新server向老client respond数据

向后兼容：老代码可以读新代码生成的新数据。或者对于service间调用，是老client向新server request数据

为什么需要向前向后兼容？
1. 大型后端应用一般都有灰度发布的需求，线上会同时存在新老版本的server
2. 客户端应用（比如ios app），用户不一定能及时更新，长期内可能新老版本共存

# Formats for Encoding Data

## Language Specific Formats
1. Java：java.io.Serializable
2. Ruby：Marshal
3. Python：pickle

pros
>allow in-memory objects to be saved and restored with minimal additional code

cons
1. 不通用：和语言绑定太深，难以与其他语言、其他系统交互
2. 反序列化进程需要具备能实例化任意class的能力。  有安全隐患
> if an attacker can get your application to decode an arbitrary byte sequence, they can instantiate arbitrary classes, which in turn often allows them to do terrible things such as remotely executing arbitrary code
3. 几乎没有版本管理，向前向后兼容的能力差
4. 编解码性能差
   >For example, Java’s built-in serialization is notorious(臭名昭著的) for its bad performance and bloated encoding


## JSON, XML, and Binary Variants

JSON、XML等存在的问题：
1. 数据类型问题、精度问题。如XML, CSV里无法区分string和number
```xml
<data>
    <number>42</number>
    <string>Hello, XML!</string>
</data>
```

JSON里无法区分integer和float，并且没有精度保证，例如：twitter用2^64数字表示twitter id。 由于JavaScript内部使用floating-point来表达数字， 后端两次返回同一个id，一次用的json number，一次用的decimal string，用于解决该问题
> integers greater than 2^53 cannot be exactly represented in an IEEE 754 double-precision floating point number

2. 不支持binary string（类似pb、thrift），只支持Unicode character strings
3. 虽然XML和JSON都有schema支持，但是使用的不多，多是在代码中hardcode编解码逻辑


## Binary Encoding
初期诞生了很多基于JSON, XML进行改造，增加支持二进制编码。
> MessagePack, BSON, BJSON, UBJSON, BISON. 
> WBXML and Fast Infoset, for example.

cons
1. 因为没有单独的schema定义字段集合，json需要把所有的field name(key)存储在数据里。本身是一种空间浪费。以MessagePack为例，实际存下来没比textual JSON短多少（81 bytes -> 66 bytes），还损失了可读性。  不太划算
   ![](/images/MessagePack.png)

## Thrift and Protocol Buffers

共同点
1. require schema
2. with a code generation tool for kinds of programming languages

Thrift第一种编码：BinaryProtocol
- type annotation: to indicate whether it is a string, integer, list, etc.
- length indication: length of a string, number of items in a list
- field tags: 对应schema里的编号。  节省很多空间。 和JSON Binary Encoding一个最大不同
![](/images/thrift-binaryprotocol.png)

Thrift第二种编码：CompactProtocol
CompactProtocol和BinaryProtocol语义上一致。  但是CompactProtocol encoding同样数据只用了34 bytes, 比BinaryProtocol（59 bytes）少了25bytes。 原因：
1. type annotation、field tag塞到了一个byte
2. 使用变长整型（整型不在固定占用8个字节，而是按照大小范围变长）存储Integer，length indication等
![](/images/thrift-compactprotocol.png)

Protocol buffer
基本和CompactProtocol一致
![](/images/protocolbuffer.png)

### Field tags and schema evolution
新增field
如果是required，new code读取old data会fail。所以为了向前兼容，新增的字段只能是optional, 或者包含default value

删除field
   - 只能删optional。 如果删除了required, 老代码读新数据会报错
   - 删除的tag number不能再次用于其他field。  否则新代码读老数据会出错

### Datatypes and schema evolution
更改数据类型：有截断风险。 如int32改为int64，向后兼容会截断

更改optional/repeated/required等属性
- pb: 中list表达为repeated。  提供了一种可能性，可以将optional改为repeated。 向后兼容，会读到list最后一个元素，多余的视为0；向前兼容，list的元素数量为0或1
- thrift: 有专门的list类型，所以没法这么改

## Avro
由于Thrift不适用于Hadoop的使用场景（原因没细说，给了ref \[21\]），Avro作为Hadoop的子项目在2009年开始研发

two schema languages
1. Avro IDL: intended for human editing
   ```c++
    record Person {
        string userName;
        union { null, long } favoriteNumber = null;
        array<string> interests;
    }
   ```
2. based on JSON: more easily machine-readable
   ```json
    {
        "type": "record",
        "name": "Person",
        "fields": [
            {"name": "userName", "type": "string"},
            {"name": "favoriteNumber", "type": ["null", "long"], "default": null},
            {"name": "interests", "type": {"type": "array", "items": "string"}}
        ]
    }
   ```

Avro和Thrift/Pb的异同
- 不同：没有tag number for field；没有data type。  （那怎么知道数据存的是啥呢？ 下一节有解释）
- 相同：也用了变长整型

![](/images/avro.png)


### The writer’s schema and the reader’s schema
> The key idea with Avro is that the writer’s schema and the reader’s schema don’t have to be the same—they only need to be compatible.

read的时候，用write schema解析，然后对比两个schema的field name，把数据从write schema 转换到 read schema。转换中，可能调整顺序、变换数据类型、空字段赋默认值等等。如图：
![](/images/avro-reader2writer-schema.png)

### Schema evolution rules

在Avro的语境下，向前向后兼容的定义是：
- forward compatibility: a new version of the schema as writer and an old version of the schema as reader
- backward compatibility:  a old version of the schema as writer and an new version of the schema as reader

**这里“前、后”，指的是前端、后端的角色。被兼容的角色是new version**
```
比如：向前兼容，schema变化的是writer，reader需要向“前”兼容变化的writer
```


>To maintain compatibility, you may only add or remove a field that has a default value.

change field data type: Avro得能支持convert不同的data type
> provided that Avro can convert the type

change field name：可以用alias，老的名字还是得在，用于匹配。  只能向后兼容

```
old schema A 
new schema B

向后兼容:
writer(old schema) A
reader(new schema) B, A remains as alias name

向前兼容:
writer(new schema) B。 除非可以带两个key的名字
reader(old schema) A
```

### But what is the writer’s schema?
某一坨来自writer的数据，reader怎么知道它对应的schema是啥？因为可能有多个数据，对应多个schema。   针对不同的场景，有不同的解法：

1. Large file with lots of records: include the writer’s schema once at the beginning of the file
2. Database with individually written records: 使用专门的database存储schema，维护版本。encoding file里带上version number
3. Sending records over a network connection: 在网络连接建立的时候，传输schema version number，然后在整个connection过程中使用该schema。 这种得每次connection传输的数据够多才有意义

### Dynamically generated schemas

以关系型数据库数据导出为例。Avro可以每次导出时按照最新的源数据schema动态生成新的schema。reader可以按照field name，识别新老数据

### Code generation and dynamically typed languages

1. 动态语言没有编译环节。生成的代码不经过编译的校验，有风险
2. Avro也为静态语言提供了code generation。基于使用场景，Avro可以从自描述（self-describing）的源文件中直接读取metadata(schema)，不一定用得到code generation。比如上面“But what is the writer’s schema?” 提到的第一种场景，schema携带在大块数据头部

## The Merits of Schemas
binary encoding based on schema相比于JSON、XML等的优势
1. 更紧凑，省空间
2. schema是比document更好的一种方式。因为要实际用于解码，所以可以保持最新状态。如果是json，靠人工document的方式记录schema，很可能逐渐失控
3. schema database的多版本管理，允许做向前向后是否兼容的check
4. code generation for static language，可以在代码编译环节校正确性、兼容性等

# Modes of Dataflow

> In the rest of this chapter we will explore some of the most common ways how data flows between processes


1. Via databases
2. Via service calls(API)
3. Via asynchronous message passing(MQ)

## Dataflow Through Databases

>preservation of unknown fields. （描述的是老代码读取新数据，且新数据新增了字段的情况）

期望：
> In this situation, the desirable behavior is usually for the old code to keep the new field intact, even though it couldn’t be interpreted

上几节提到的各种encoding formats是可以实现这种期望的。但是有一点要特别注意：避免在application内部的object转换中，丢掉new field，这个环节是不受database encoding formats控制的
![](/images/preservation-unknow-field.png)

### Different values written at different times
schema change（比如加字段）的时候，如果老数据完全改换新的schema，数据全刷一遍的成本太高。  因此都是DB底层做兼容。 例如read老数据的时候，对不存在的column赋予默认值
（这里想起了公司内系统的各种重构，以及和数据仓库协同升级的方式）

schema evolution使得schema展示唯一，但实际上底层的数据因时间而有差异。  DB做了中间的兼容（adapter）

### Archival storage
dump数据的各种场景（如snapshot、analytic warehouse等），即使源数据存在多个版本的encoding，在dump的时候一律按照latest schema dump。  因为反正要全扫一遍

## Dataflow Through Services: REST and RPC
### Web services
比较了下REST和SOAP

### The problems with remote procedure calls (RPCs)
RPC本身的定义想将网络请求封装成和本地函数调用一样丝滑。但实际上和本地函数调用还是有很多不一样的地方：书中列了6点。 因此，没有必要非得把remote service和本地function调用搞到一起，本质上就是两件事

### Current directions for RPC
1. gRPC: using Protocol Buffers
2. Finagle: using Thrift
3. Rest.li: using JSON over HTTP

总结来，REST主宰了API； RPC主要用于同组织内不同service间的请求


### Data encoding and evolution for RPC
一般都是server先更新，client后更新。从API的角度，向前向后兼容对应于：
1. 向前：老client解读新server的respond的新API数据
2. 向后：新server接收老client request的老API的数据

RPC的向前向后兼容能力，来自其使用的encoding formats

API的版本管理：
1. RESTful中，在URL或者HTTP Accept header使用version number维护版本
2. 对于一些使用API密钥来鉴别特定client的服务，通常把版本存在server，经由单独的接口来管理版本


## Message-Passing Dataflow
介绍了下面内容，比较常识，就不细总结了
1. MQ相比于RPC的好处
2. MQ的基本设定
3. 一种分布式并发控制的框架- Distributed actor framework。 也用到了MQ，来实现位置透明





   
