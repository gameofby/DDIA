1. 向前兼容：新代码可以读老代码生成的老数据
2. 向后兼容：老代码可以读新代码生成的新数据

为什么需要向前向后兼容？
1. 大型后端应用一般都有灰度发布的需求
2. 客户端应用，用户不一定能及时更新，长期内可能新老版本共存

# Formats for Encoding Data

## Language Specific Formats
1. Java：java.io.Serializable
2. Ruby：Marshal
3. Python：pickle

优势
>allow in-memory objects to be saved and restored with minimal additional code

劣势
1. 和语言绑定太深，难以与其他语言、其他系统交互
2. 反序列化进程需要具备能实例化任意class的能力。  有安全隐患
3. 几乎没有版本管理，向前向后兼容的能力
4. 编解码性能差
   >For example, Java’s built-in serialization is notorious for its bad performance and bloated encoding


## JSON, XML, and Binary Variants

JSON、XML等存在的问题：
1. 数据类型问题、精度问题:
   - 如XML里无法区分string和number； 
   - JSON里无法区分integer和float，并且没有精度保证，例如：twitter用2^64数字表示twitter id。 由于JavaScript内部使用floating-point来表达数字， 后端两次返回同一个id，一次用的json number，一次用的decimal string，用于解决改问题
    >integers greater than 2^53 cannot be exactly represented in an IEEE 754 double-precision floating point number
2. 不支持binary string（类似pb、thrift），只支持Unicode character strings
3. 虽然XML和JSON都有schema支持，但是使用的不多，多是在代码中hardcode编解码逻辑


## Binary Encoding
即直接对JSON等进行二进制编码，还是依附于JSON存在的

劣势
1. 因为没有单独的schema，json需要把所有的field name(key)存储在数据里。本身是一种空间浪费。以MessagePack为例，实际存下来没比textual JSON短多少（81 bytes -> 66 bytes），还损失了可读性。  不太划算
   ![](/images/MessagePack.png)

## Thrift and Protocol Buffers

共同点
1. require schema
2. with a code generation tool for kinds of languages

Thrift第一种编码：BinaryProtocol
1. type annotation: to indicate whether it is astring, integer, list, etc.
2. length indication: length of a string, number of items in a list
3. field tags: 对应schema里的编号。  节省很多空间。 和JSON Binary Encoding一个最大不同
![](/images/thrift-binaryprotocol.png)

Thrift第一种编码：CompactProtocol
CompactProtocol和BinaryProtocol语义上一致。  但是前者encoding同样数据只用了34 bytes。 原因：
1. type annotation、field tag塞到了一个byte
2. 使用变长整型（整型不在固定占用8个字节，而是按照大小范围变长）存储Integer，length indication等
![](/images/thrift-compactprotocol.png)

Protocol buffer
基本和CompactProtocol一致
![](/images/protocolbuffer.png)


   
