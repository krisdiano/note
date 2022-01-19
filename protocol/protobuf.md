# proto encoding rules

## base encoding

### base 128 varint

`varint`是使用一个或多个字节序列化整数的方法，较小的数字占用较少数量的字节。在`varint`中一个字节的`msb`不存储数据，而是标识是否还有后续字节。多个字节时之间的关系为小端字节序。

```bash
dec : 300
bin : 00000001 00101100
reverse : 00101100 00000001
split : 10101100 00000010
```

### message structure

`message`只是一系列的键值对，序列化后并没有`key`，而是使用`field num`标识`key`，因此解码时必须知道`key`和`field num`的映射关系。

> 信息=数据+上下文

在这里上下文就是类型信息，当前使用的类型信息如下：

| type | meaning          | used for                                            |
| ---- | ---------------- | --------------------------------------------------- |
| 0    | varint           | int32,int64,uint32,uint64,sint32,sint64,bool,enum   |
| 1    | 64-bit           | fixed64,sfixed64,double                             |
| 2    | Length-delimited | string,bytes,embedded messages,packed repeat fields |
| 3    | Start group      | groups(deprecated)                                  |
| 4    | End group        | groups(deprecated)                                  |
| 5    | 32-bit           | fixed32,sfixed32,float                              |

`key = tag << 3 | wire_type`，公式中的`tag`就是定义数据模式时的`field num`，`key`对应的编码规则是固定的，采用`varint`编码。

```bash
# src
message Test {
	optional int32 a = 1;
}

# dst 
0x08 0x96 0x01
# decode
0x08 => tag:1 wire_type:0(varint) # a的field num是1，值的部分是varint编码
# decode data
0x96 0x01 => 0x01 0x96 => 0000001 0010110 => 10010110 => 150 # a的值被设为150
```

### zigzag

计算机对数字的处理使用的是二进制的补码形式， 每一个负数都有一个补码与之相同的正数，且负数越小，正数越大，正式如此，变量被设置为一个很小的负整数时，其实对应的是一个很大的正整数，那么`varint`就失去了优势，为了解决这个问题，就需要引入`zigzag`编码。

`ZigZag`的思想是将小于0的整数映射为一个大于0的整数，映射的规则为：

| 类型   | 源   | 目的     |
| ------ | ---- | -------- |
| 0      | 0    | 0        |
| 正整数 | n    | 2n       |
| 负整数 | n    | \|2n\|-1 |

根据映射规则，可以得到一个结论：`zigzag`和`varint`结合使用可以确保绝对值小的数字占用的空间较少。

### Non-varint Numbers

在某些场景下，可以确定数字的值基本都是很大的，`varint`已经没有优势的情况下，可以选择合适的类型，这些类型在编码时，直接使用定长的编码。

### length delimited

> kv = key length value

`length`与`key`相同，采用`varint`编码。

```bash
# src 
message Test2 {
  optional string b = 2;
}

# dst
0x12 0x07 0x74 0x65 0x73 0x74 0x69 0x6e 0x67
# decode
0x12 => tag:2 wire_type:2(length delimited) # b的field num是2，值的部分是length delimited编码
# docode length
deocde_varint(0x07) => 7 # 后续的7个字节是值的序列化内容
# docode data
0x74 0x65 0x73 0x74 0x69 0x6e 0x67 # 对应到string类型后，b的值被设置为testing
```

### field order

在`proto`中如何定义一个`message`内的成员之间的`field num`的顺序不会影响序列化，在序列化时也不会保证已知字段和未知字段的写入顺序，这个实现是很细节的层面，将来可能会改变内部实现，因此解析器必须能够以任何顺序解析字段。

## 疑问

只有`field`是有符号数据类型，且不是定长时，采用`varint`和`zigzag`混合编码？`length delimited`中标识长度的部分和`key`一样只采用`varint`编码？



## Q&A

在了解编码的基本知识后，会更加理解一些问题。

### 为什么常用的字段的`field num`使用[0,15]

因为`field num`和`wire_type`是混在一起的，而且这个部分使用`varint`编码，因此`msb`不能存储数据，一个字节的时候只有7`bits`存储数据，`wire_type`占用了3`bits`，只有4`bits`可以存储`field num`，对应的数字范围就是`[0,15]`，超过15的部分就会使用两个及以上字节。

### 为什么`proto`中提供了`reserved fields`

协议在使用一段时间后，可能持久化了很多序列化之后的内容，如果期间消息结构升级，删除一些字段时，如果没有把删除的字段保留到`reserved`中，那么后续重用这些字段时，反序列化可能会失败。因为序列化内容中的关于`key`的部分并不会带有`field name`的信息，而仅仅有`field num`的信息。

至于为什么`reserved`支持`field name`，这个还需要研究是怎么做的。

### 为什么有这么多数字的类型

需要结合业务场景和编码类型选定合适的类型，可以把空间利用率最大化。



