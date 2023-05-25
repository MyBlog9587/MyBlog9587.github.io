---
title: grpc笔记
date: 2023-05-24 15:33:20
categories: grpc
tags:
- grpc
- private
- protobuf
---



# protocol buffers 

> 版本：proto3

## 格式
```protobuf
syntax = "proto3";

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 result_per_page = 3;
}

message xxx {
    // 字段规则：required -> 字段只能也必须出现 1 次
    // 字段规则：optional -> 字段可出现 0 次或多次
    // 字段规则：repeated -> 字段可出现任意多次（包括 0）
    // 类型：int32、int64、sint32、sint64、string、32-bit ....
    // 字段编号：0 ~ 536870911（除去 19000 到 19999 之间的数字）
    字段规则 类型 名称 = 字段编号;
}
```

### **字段规则：**

- required: 字段只能也必须出现 1 次
  proto3中已经不使用了，变成默认项
- optional: 字段可出现 0 次或多次
  可选字段处于两种可能的状态之一：
  - 字段已设置，并包含了明确设置或从传输线解析的值。它将被序列化到传输线上。
  - 字段未设置，并将返回默认值。它不会被序列化到传输线上
- repeated: 字段可出现任意多次（包括 0）
  这种字段类型在格式良好的消息中可以重复零次或多次。重复值的顺序将被保留。
- map: 这是一种键值对字段类型



### 字段类型对照表：

| .proto 类型 | 备注 | C++ 类型 | Java/Kotlin 类型[1] | Python 类型[3] | Go 类型 | Ruby 类型 | C# 类型 | PHP 类型 | Dart 类型 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| double | 双精度浮点数 | double | float | float64 | Float | double | float | double |
| float | 单精度浮点数 | float | float | float32 | Float | float | float | double |
| int32 | 使用可变长度编码。对于编码负数效率低下——如果字段可能包含负值，请使用sint32代替。 | int32 | int | int | int32 | Fixnum或Bignum（根据需要） | int | integer | int |
| int64 | 使用可变长度编码。对于编码负数效率低下——如果字段可能包含负值，请使用sint64代替。 | int64 | long | int/long[4] | int64 | Bignum | long | integer/string[6] | Int64 |
| uint32 | 使用可变长度编码。 | uint32 | int[2] | int/long[4] | uint32 | Fixnum或Bignum（根据需要） | uint | integer | int |
| uint64 | 使用可变长度编码。 | uint64 | long[2] | int/long[4] | uint64 | Bignum | ulong | integer/string[6] | Int64 |
| sint32 | 使用可变长度编码。有符号整数值。相比普通的int32，更有效地编码负数。 | int32 | int | int | int32 | Fixnum或Bignum（根据需要） | int | integer | int |
| sint64 | 使用可变长度编码。有符号整数值。相比普通的int64，更有效地编码负数。 | int64 | long | int/long[4] | int64 | Bignum | long | integer/string[6] | Int64 |
| fixed32 | 固定长度为4字节。如果值通常大于2^28，比uint32更高效。 | uint32 | int[2] | int/long[4] | uint32 | Fixnum或Bignum（根据需要） | uint | integer | int |
| fixed64 | 固定长度为8字节。如果值通常大于2^56，比uint64更高效。 | uint64 | long[2] | int/long[4] | uint64 | Bignum | ulong | integer/string[6] | Int64 |
| sfixed32 | 固定长度为4字节。 | int32 | int | int | int32 | Fixnum或Bignum（根据需要） | int | integer | int |
| sfixed64 | 固定长度为8字节。 | int64 | long | int/long[4] | int64 | Bignum | long | integer/string[6] | Int64 |
| bool | 布尔值 | boolean | bool | bool | TrueClass/FalseClass | bool | boolean | bool |
| string | 字符串必须始终包含UTF-8编码或7位ASCII文本，且长度不能超过2^32 | string | String | str/unicode[5] | string | String (UTF-8) | string | String | |
| bytes | 可以包含不超过2^32的任意字节序列 | string | ByteString | str (Python 2) | bytes (Python 3) | | | | |


## 字段默认值：
解析消息时，如果编码消息不包含特定的单数元素，则解析对象中的相应字段将设置为该字段的默认值。这些默认值是特定于类型的：

- 对于字符串，默认值为空字符串。
- 对于字节，默认值为空字节。
- 对于布尔值，默认值为 false。
- 对于数字类型，默认值为零。
- 对于枚举，默认值是第一个定义的枚举值，它必须是 0。
- 对于消息字段，该字段未设置。它的确切值取决于语言。

重复字段的默认值为空


## 枚举值：
希望其字段之一仅具有预定义值列表之一

```proto
enum Corpus {
  CORPUS_UNSPECIFIED = 0;
  CORPUS_UNIVERSAL = 1;
  CORPUS_WEB = 2;
  CORPUS_IMAGES = 3;
  CORPUS_LOCAL = 4;
  CORPUS_NEWS = 5;
  CORPUS_PRODUCTS = 6;
  CORPUS_VIDEO = 7;
}

message SearchRequest {
  string query = 1;
  int32 page_number = 2;
  int32 results_per_page = 3;
  Corpus corpus = 4;
}
```
每个枚举定义都必须包含一个映射到零的常量作为其第一个元素。这是因为：
- 必须有一个零值，这样我们就可以使用 0 作为数字 默认值。
- 零值需要是第一个元素，以便与 第一个枚举值始终是默认值的proto2语义兼容。



## 嵌套类型：
```proto
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}
```

如果要在其父消息类型之外重用此消息类型，请将其称为_Parent_._Type_：

```proto
message SomeOtherMessage {
  SearchResponse.Result result = 1;
}
```

## Any 类型：
Any消息类型允许在**不需要其.proto定义**的情况下使用消息作为嵌套类型。Any包含一个任意序列化的消息字节流，以及一个充当全局唯一标识符并解析为该消息类型的URL。要使用Any类型，需要导入 `google/protobuf/any.proto` 文件

```proto
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

## Oneof 类型：
如果有一个包含许多字段的消息，并且同时最多只能设置一个字段，您可以通过使用oneof功能来强制实现此行为并节省内存。

oneof字段与普通字段类似，只是在一个oneof中的所有字段共享内存，并且同时最多只能设置一个字段。设置oneof的任何成员将自动清除所有其他成员。

```proto
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

> 注意，如果设置了多个值，则根据proto中的顺序，最后设置的值将覆盖所有先前的值。


## map

```proto
map<key_type, value_type> map_field = N;
```
其中，key_type可以是任何整数或字符串类型（即，任何标量类型，除了浮点类型和字节类型）。请注意，枚举类型不是有效的key_type。value_type可以是除了另一个map之外的任何类型。

- Map字段不能重复。
- 映射值的线路格式顺序和映射迭代顺序是未定义的，因此不能依赖于映射项以特定顺序出现。
- 在生成.proto文件的文本格式时，映射按键排序。数字键按数字顺序排序。
- 从线路解析或合并时，如果存在重复的映射键，则使用最后一个出现的键。从文本格式解析映射时，如果存在重复键，则解析可能失败。
- 如果为映射字段提供了键但没有值，在序列化字段时的行为取决于编程语言。在C++、Java、Kotlin和Python中，将序列化类型的默认值，而在其他语言中不会序列化任何内容。




第一行指定使用的是 `proto3` 语法：如果不指定，协议缓冲区编译器默认使用的是proto2。这必须是文件的第一个非空、非注释行

`SearchRequest` 定义数据格式, 消息定义中的每个字段都有一个唯一的编号;

> 请注意，1 到 15 范围内的字段编号需要一个字节进行编码，包括字段编号和字段类型（您可以在Protocol Buffer Encoding中找到更多相关信息）。16 到 2047 范围内的字段编号占用两个字节。因此，您应该为非常频繁出现的消息元素保留数字 1 到 15。请记住为将来可能添加的频繁出现的元素留出一些空间。

**保留字段**: (reserved)
指定已删除字段的字段号或名称

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

不能在同一语句中混合字段名称和数值


### [类型对照表](https://developers.google.com/protocol-buffers/docs/proto3#scalar)

### 默认值

| 类型 | 默认值|
|  ----  | ----  |
| string | 空 |
| bytes | 空 |
| bools | false |
| 数值 | 0 |
| 枚举值 | 0 |
| message  | 未设置该字段。它的确切值取决于语言 |

> 重复字段的默认值为空

解析以后的值 无法区分是否为默认值. 如果字段设置了默认值，那么该值不会被序列化

### oneof 
```protobuf
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

oneof 字段和普通字段一样，只是一个 oneof **共享**内存中的所有字段，最多可以同时设置一个字段。设置 oneof 的任何成员会自动清除所有其他成员,即只有最后一个生效.


### maps
```protobuf
map<key_type, value_type> map_field = N;
```

### Packages
可以在`.proto`文件中添加一个可选的包说明符，以**防止协议消息类型之间的名称冲突**, 然后可以在定义你的消息类型的字段时使用包说明符.

```protobuf
package foo.bar;
message Open { ... }

message Foo {
  ...
  foo.bar.Open open = 1;
  ...
}
```


### rpc 服务定义
想在RPC(远程过程调用)系统中使用消息类型，需要在`.proto`文件中定义RPC**服务接口**
```protobuf
service SearchService {
  rpc Search(SearchRequest) returns (SearchResponse);
}
```

### [json 映射](https://developers.google.com/protocol-buffers/docs/proto3#json)
Proto3支持JSON中的规范编码，使得在系统之间共享数据更加容易

如果json编码的数据中缺少一个值，或者它的值为空，那么在解析到协议缓冲区时，它将被解释为适当的默认值。如果一个字段在协议缓冲区中有默认值，它将在json编码的数据中被默认省略，以节省空间。实现可以提供选项，在json编码的输出中发出带有默认值的字段

## 生成pb消息的对应代码
```bash
protoc --proto_path=IMPORT_PATH --go_out=DST_DIR path/to/file.proto
```

### 参数说明：
- `IMPORT_PATH` 指定 `.proto` 解析`import`指令时在其中查找文件的目录。如果省略，则使用当前目录。
- `--proto_path` 多次传递该选项可以指定多个导入目录；他们将被按顺序搜索, `-I=_IMPORT_PATH_` 可以用作 的简写形式 `--proto_path`
- `DST_DIR` 生成代码路径


### 输出指令：
- `--cpp_out`生成 C++ 代码DST_DIR
- `--java_out`生成 Java 代码DST_DIR
- `--kotlin_out`在DST_DIR
- `--python_out`生成 Python 代码DST_DIR
- `--go_out`生成 Go 代码DST_DIR
- `--ruby_out`生成 Ruby 代码DST_DIR
- `--objc_out`在DST_DIR
- `--csharp_out`生成 C# 代码DST_DIR
- `--php_out`生成 PHP 代码DST_DIR



## 规则
- 包名应该是小写的 
- 消息名称首字母大写 `message SongServerRequest`
- 如果字段名称包含数字，则该数字应出现在字母之后而不是下划线之后。例如，使用`song_name1` 代替 `song_name_1`
- 重复的字段使用复数名称
- 枚举类型名称使用 CamelCase（首字母大写）, 每个枚举值都应该以分号结尾，而不是逗号; 零值枚举应该有后缀 `UNSPECIFIED`
- `RPC` 服务，应该使用 CamelCase（首字母大写）作为服务名称和任何 `RPC` 方法名称

    ```protobuf
    service FooService {
       rpc GetSomething(GetSomethingRequest) returns (GetSomethingResponse);
       rpc ListSomething(ListSomethingRequest) returns (ListSomethingResponse);
    }
    ```


## 编码
### Varints: 
是一种使用一个或多个字节序列化整数的方法。较小的数字占用较少的字节数。

除了最后一个字节外，`varint` 中的每个字节都设置了**最高有效位**（msb）-这**表明还有进一步的字节**。每个字节的低7位用于以7位为一组存储数字的二进制**补码**表示，**最低有效组在前**

#### 举例：
300

```bash
1010 1100 0000 0010
```

如何确定这是300？首先，从每个字节中删除msb，因为这是在告诉我们是否已到达数字的末尾：

```bash
 1010 1100 0000 0010
→ 010 1100  000 0010
```

反转两组7位，因为varint存储数字时最低有效组在前
```bash
   010 1100 000 0010
→  000 0010 010 1100
```

```bash
   000 0010  010 1100
→  000 0010 ++ 010 1100
→  10 010 1100
→  2^8 + 2^5 + 2^3 + 2^2
→  256 + 32 + 8 + 4 = 300
```

## 解码
protobuf 消息是一系列**键-值**对。消息的二进制版本只使用字段的数字作为键——每个字段的名称和声明的类型只能在解码端通过引用 protobuf 消息类型的定义(即.proto文件)来确定。

在对消息进行解码时，解析器需要能够跳过它无法识别的字段。通过这种方式，可以将新字段添加到消息中，而不会破坏不知道这些字段的旧程序。为此，段格式消息中每一对的“键”实际上是两个值——来自`.proto`文件的字段号，加上一个段类型，该类型提供了足够的信息来查找以下值的长度。

### **段类型**(wire_type)：

|类型|代表值|作用范围|
| ---- | ---- | ---- |
|0|Varint|int32, int64, uint32, uint64, sint32, sint64, bool, enum|
|1|64 位|fixed64, sfixed64, double|
|2|Length-delimited|string, bytes、嵌入消息、打包的重复字段|
|3|开始组|分组（已弃用）|
|4|结束组|分组（已弃用）|
|5|32 位|fixed32, sfixed32, float|

流式消息中的每个键都是一个 `varint`，其值为 `(field_number << 3) | wire_type` ——换句话说，**数字的最后三位存储段类型**

### 无符号数值型
#### 举例:
消息如下： 150 序列化后会得到3个字节（16进制）
`08 96 01`

流中的第一个数字始终是 `varint` 键，这里是 08 或者（删除 msb）.

其中第一个字节（08）是一个 varint（每个字节的最高位 = 1 表示该 int 还需要拼上后续字节的低 7 bits），其内容包含了第一个元素的序号（field number）和类型（wire type）

将 08 的二进制 `0000 1000` 拆分成三部分来解释：
```bash
0000 1000
```
 - `0`: 表示这个 varint 到这个字节就结束了
 - `0001`: 表示其序号是 1
 - `000`: 表示其值类型也是个 `varint`

即 取最后三位来获得段类型(0)，然后右移3以获得字段号(1)。因此，现在知道字段号是1，下面的值是一个 `varint`

```bash
  96 01
→ 1001 0110  0000 0001
→ 000 0001  ++  001 0110 (drop the msb and reverse the groups of 7 bits)
→ 1 0010110
→ 128 + 16 + 4 + 2 = 150
```

### 有符号数值型
与段类型(write-type) `0` 关联的所有协议缓冲区类型都被编码为 `varint`。 但是，在编码负数时，带符号的 int 类型（sint32 和 sint64）与"标准" int 类型（int32 和 int64）之间存在重要区别。

如果使用 int32 或 int64 作为负数的类型，则生成的 varint 始终为 10 个字节长——实际上，它被视为一个非常大的无符号整数。 如果您使用其中一种有符号类型，则生成的 varint 将使用 ZigZag 编码，这种编码效率更高。

ZigZag编码将有符号整数映射为无符号整数，因此具有小绝对值的数字(例如，-1)也具有小的varint编码值。它的方式是在正整数和负整数之间来回“之字形”，因此-1被编码为1,1被编码为2，-2被编码为3

```bash
(n << 1) ^ (n >> 31) // sint32s
(n << 1) ^ (n >> 63) // 64-bit 
```

### 非 varint 数值型
非 `varint` 数值类型很简单——`double` 和 `fixed64` 的段类型为 `1`，它告诉解析器期待一个固定的 64 位数据块； 类似地，`float` 和 `fixed32` 的段类型为 `5`，它告诉它期望 32 位。 在这两种情况下，值都以小端字节顺序存储


### string
#### 举例：
```protobuf
message Test2 {
  optional string b = 2;
}
```

将 `b` 的值设置为`"testing"`会得到
```bash
12 07 [74 65 73 74 69 6e 67]
```

> 括号中的字节是“testing”的UTF8

关键字 `0x12`
```bash
0x12
→ 0001 0010  (binary representation)
→ 00010 010  (regroup bits)
→ field_number = 2, wire_type = 2
```

字段序号= 2, wire类型 = 2. 值的长度 varint 是7（`0x07`）, 即[] 内的数据


#### 嵌入式

```protobuf
message Test1 {
  optional int32 a = 1;
}

message Test3 {
  optional Test1 c = 3;
}
```
Test1 的a字段再次设置为 150 后得编码：
```bash
 1a 03 08 96 01
 0001 1010 // 0x1a
 0000 0011 // 0x03
```

根据`0x1a` 可以算出 段类型(wire type) = 2， 并且数据长度为 `0x03` = 3,即 `08 96 01` = 150



# grpc
- 使用 HTTP/2 来进行传输
- 使用 protocol buffers 作为 IDL 来定义服务接口

## 安装gRPC库
`go get -u google.golang.org/grpc`  默认安装版本太低

`go install google.golang.org/protobuf/cmd/protoc-gen-go@latest`

`go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest`

## 安装GO 语言protoc插件
`go get -u github.com/golang/protobuf/protoc-gen-go` 

如果报异常：`go: module github.com/golang/protobuf is deprecated: Use the "google.golang.org/protobuf" module instead.` 就使用下面方式安装（go 版本问题）

`go install google.golang.org/protobuf/cmd/protoc-gen-go@latest`

> 看下非GOPATH路径下 是否可执行，也许需要拷贝到系统目录 才生效

## 使用举例
### 创建数据接口以及接口
orderinfo.proto
```protobuf
// 注意都要使用 ; 结尾
// 指定pb 使用版本
syntax = "proto3";

// 定义包命
// 如果 go_package 指定了name， 此处就需要省略。否则会提示异常：WARNING: Missing 'go_package' option in "orderinfo.proto"
// package orderinfo;

// 如果 使用了 packeage 那么 go_package 不能指定name 选项，冲突
// option go_package = ".";
option go_package = ".;orderinfo";

// 使用 message 创建消息格式
message Info_data {
  // 注意结束使用 ; 
  string id = 1;
  string name = 2;
  string describe = 3;
}

message Info_ID {
  string price = 1;
}

// 定义函数接口
// 必须使用关键字 service 定义
service Info {
  // 必须使用 rpc 关键字声明, 注意返回 使用 returns
  rpc addInfo(Info_data) returns (Info_ID);
  rpc getInfo(Info_ID) returns (Info_data);
}
```

> option go_package = "path;name";  
path: 表示生成go文件的存放地址，会自动生成目录; 注意跟protpc 执行是的目录指定关系（否则会出现在指定路径下又创建一层path）
name: 表示生成go文件所属的包名

### 生成对应语言库
`protoc -I=plugins --go_out=orderinfo --go-grpc_out=orderinfo plugins/orderinfo.proto`

`protoc -I=$SRC_DIR --go_out=$DST_DIR $SRC_DIR/addressbook.proto`

windows 下:
`protoc -I E:\grpc-note\example\client_server_rpc\server\proto --go_out=E:\grpc-note\example\client_server_rpc\server\ecommerce --go-grpc_out=E:\grpc-note\example\client_rpc\server\ec
ommerce E:\grpc-note\example\client_server_rpc\server\proto\ecommerce.proto`

```bash
# tree 
.
├── orderinfo
│   ├── orderinfo.pb.go
│   └── orderinfo_grpc.pb.go
└── plugins
    └── orderinfo.proto
```

### 库样例：

orderinfo/orderinfo.pb.go:

```go
//
// 指定pb 使用版本

// Code generated by protoc-gen-go. DO NOT EDIT.
// versions:
//      protoc-gen-go v1.24.0
//      protoc        v3.5.0
// source: orderinfo.proto

package orderinfo

import (
        proto "github.com/golang/protobuf/proto"
        protoreflect "google.golang.org/protobuf/reflect/protoreflect"
        protoimpl "google.golang.org/protobuf/runtime/protoimpl"
        reflect "reflect"
        sync "sync"
)

const (
        // Verify that this generated code is sufficiently up-to-date.
        _ = protoimpl.EnforceVersion(20 - protoimpl.MinVersion)
        // Verify that runtime/protoimpl is sufficiently up-to-date.
        _ = protoimpl.EnforceVersion(protoimpl.MaxVersion - 20)
)

// This is a compile-time assertion that a sufficiently up-to-date version
// of the legacy proto package is being used.
const _ = proto.ProtoPackageIsVersion4

// 创建消息格式
type InfoData struct {
        state         protoimpl.MessageState
        sizeCache     protoimpl.SizeCache
        unknownFields protoimpl.UnknownFields

        // 注意结束使用 ;
        Id       string `protobuf:"bytes,1,opt,name=id,proto3" json:"id,omitempty"`
        Name     string `protobuf:"bytes,2,opt,name=name,proto3" json:"name,omitempty"`
        Describe string `protobuf:"bytes,3,opt,name=describe,proto3" json:"describe,omitempty"`
        ...
```

> proto文件中的注释会被生成到对应的库里


### 添加服务端代码
```go
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc/codes"
	pb "grpc_server/orderinfo"
	"log"
	"net"

	"github.com/gofrs/uuid"
	"google.golang.org/grpc"
	"google.golang.org/grpc/status"
)

const port = ":2023"

// 用来实现 orderinfo/orserinfo 的服务器
type server struct {
	orderinfoMap map[string]*pb.InfoData
}

// 实现 orderinfo.AddInfo 的 AddInfo方法
func (s *server) AddInfo( ctx context.Context, data *pb.InfoData) (*pb.Info_ID, error) {
	out, err := uuid.NewV4()
	if err != nil {
		return nil, fmt.Errorf("Error: make id ", err)
	}

	data.Id = out.String()

	if s.orderinfoMap == nil {
		s.orderinfoMap = make(map[string]*pb.InfoData)
	}

	s.orderinfoMap[data.Id] = data

	return &pb.Info_ID{Price: data.Id}, status.New(codes.OK, "").Err()
}


func (s *server) GetInfo(ctx context.Context, id *pb.Info_ID) (*pb.InfoData, error) {
	if s.orderinfoMap == nil {
		return nil, fmt.Errorf("no data")
	}

	data, exists := s.orderinfoMap[id.Price]
	if !exists {
		return nil, status.Errorf(codes.NotFound, "id does not exist", id.Price)
	}
	return data, status.New(codes.OK, "").Err()
}

func main()  {
	// 建立server 监听
	conn, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("socket listen : %v", err)
	}

	// 创建grpc server 端
	s := grpc.NewServer()
	// 注册grpc
    pb.RegisterInfoServer(s, &server{})
	// 接收等待
	if err := s.Serve(conn); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

client.go 
```go
package main

import (
	"context"
	"fmt"
	"google.golang.org/grpc"
	pb "grpc_client/orderinfo"
	"log"
	"time"
)

const port = ":2023"

func main() {
	// 建立server连接
	conn, err := grpc.Dial(port, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("don't connect server: %v", err)
	}
	defer conn.Close()

	// 构建测试数据
	id := "1"
	name := "test one"
	description := "第一个测试项目"
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	defer cancel()

	// 构建 grpc client 端
	c := pb.NewInfoClient(conn)

	// 测试add 函数
	r, err := c.AddInfo(ctx, &pb.InfoData{Id: id, Name: name, Describe: description})
	if err != nil {
		log.Fatalf("add data err: %v", err)
	}
	fmt.Printf("add data price : %v\n", r.Price)

	// 测试 get 函数
	d, err := c.GetInfo(ctx, &pb.InfoID{Price: r.Price})
	if err != nil {
		log.Fatalf("get data err: %v", err)
	}
	fmt.Printf("get data info: %v\n", d.String())

	fmt.Println("grpc test end!")
}
```

> 上述测试用例 是简单的请求-响应模式

## grpc 通信模式
用 `stream` 类型来指定流式传输方法, 用来区分哪种通信模式

- 一元 RPC 

    ```protobuf
    rpc GetFeature(Point) returns (Feature) {}
    ```

- 服务器端流 RPC

    ```protobuf
    rpc ListFeatures(Rectangle) returns (stream Feature) {}
    ```

- 客户端流 RPC

    ```protobuf
    rpc RecordRoute(stream Point) returns (RouteSummary) {}
    ```

- 双向流 RPC

    ```protobuf
    rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}
    ```



### 一元 RPC

当客户端调用服务器端的远程方法时，**客户端发送请求至服务器端并获得一个响应**，与响应一起发送的还有状态细节以及 trailer 元数据。

在一元 RPC 模式中， gRPC 服务器端和 gRPC 客户端在通信时始终只有一个请求和一个响应。例如上面的测试用例。

具体示例代码参考[unitary_rpc](./example/unitary_rpc)

**demo样例:**

```go
func (s *routeGuideServer) GetFeature(ctx context.Context, point *pb.Point) (*pb.Feature, error) {
  for _, feature := range s.savedFeatures {
    if proto.Equal(feature.Location, point) {
      return feature, nil
    }
  }
  // No feature was found, return an unnamed feature
  return &pb.Feature{Location: point}, nil
}
```

<img src="./img/一元RPC.png">

### 服务器端流 RPC 模式
在服务器端流 RPC 模式中， 服务器端在接收到客户端的请求消息后，会返回一个响应的序列。 这种**多个**响应所组成的序列也被称为“流”。 在将所有的服务器端响应发送完毕之后， 服务器端会以 trailer 元数据的形式将其状态发送给客户端， 从而标记流的结束。

即 客户端发出一个请求之后， 会接收到多条响应消息。

具体示例代码参考[server_rpc](./example/server_rpc)

**demo样例:**

```go
func (s *routeGuideServer) ListFeatures(rect *pb.Rectangle, stream pb.RouteGuide_ListFeaturesServer) error {
  for _, feature := range s.savedFeatures {
    if inRange(feature.Location, rect) {
      if err := stream.Send(feature); err != nil {
        return err
      }
    }
  }
  return nil
}
```

<img src="./img/服务端RPC.png">


### 客户端流RPC模式
客户端会发送**多个请求**给服务器端， 而不再是单个请求。 服务器端则会发送**一个响应**给客户端。 但是， 服务器端不一定要等到从客户端接收到所有消息后才发送响应。

在服务器端流 RPC 模式代码基础上进行添加 update 方法, 更新pb文件 重新生成客户端、服务器代码

具体示例代码参考[client_rpc](./example/client_rpc)
<img src="./img/客户端流RPC.png">


### 双向流RPC模式
客户端以消息流的形式发送请求到服务器端，服务器端也以消息流的形式进行响应。 调用必须由客户端发起

具体示例代码参考[client_server_rpc](./example/client_server_rpc)
<img src="./img/双向流RPC.png">


## 验证的几种模式

### 1. 基本情况-没有加密或认证

Client: 

```go
conn, _ := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(insecure.NewCredentials()))
// error handling omitted
client := pb.NewGreeterClient(conn)
// ...
```

Server:

```go
s := grpc.NewServer()
lis, _ := net.Listen("tcp", "localhost:50051")
// error handling omitted
s.Serve(lis)
```

>   grpc.WithTransportCredentials(insecure.NewCredentials())

### 2. 使用服务器身份验证 SSL/TLS

Client:

```go
creds, _ := credentials.NewClientTLSFromFile(certFile, "")
conn, _ := grpc.Dial("localhost:50051", grpc.WithTransportCredentials(creds))
// error handling omitted
client := pb.NewGreeterClient(conn)
// ...
```

Server:

```go
creds, _ := credentials.NewServerTLSFromFile(certFile, keyFile)
s := grpc.NewServer(grpc.Creds(creds))
lis, _ := net.Listen("tcp", "localhost:50051")
// error handling omitted
s.Serve(lis)
```

>   credentials.NewClientTLSFromFile

### 3. 使用谷歌进行身份验证

```go
pool, _ := x509.SystemCertPool()
// error handling omitted
creds := credentials.NewClientTLSFromCert(pool, "")
perRPC, _ := oauth.NewServiceAccountFromFile("service-account.json", scope)
conn, _ := grpc.Dial(
	"greeter.googleapis.com",
	grpc.WithTransportCredentials(creds),
	grpc.WithPerRPCCredentials(perRPC),
)
// error handling omitted
client := pb.NewGreeterClient(conn)
// ...
```



## 启动方式

### server：

```go
flag.Parse()
// 侦听客户端请求的端口
lis, err := net.Listen("tcp", fmt.Sprintf("localhost:%d", *port))
if err != nil {
  log.Fatalf("failed to listen: %v", err)
}
var opts []grpc.ServerOption
...
// 创建 gRPC 服务器的实例
grpcServer := grpc.NewServer(opts...)
// 向 gRPC 服务器注册服务实现
pb.RegisterRouteGuideServer(grpcServer, newServer())
// 调用Serve()服务器进行阻塞等待，直到进程被终止或被Stop()调用
grpcServer.Serve(lis)
```

### client：

```go
var opts []grpc.DialOption
...
// 设置 gRPC 通道: 将服务器地址和端口号传递给 grpc.Dial()如下来创建它
// Dial 不需要任何凭据
conn, err := grpc.Dial(*serverAddr, opts...)
// 也可以使用 DialOptions 来设置身份验证凭据
if err != nil {
  ...
}
defer conn.Close()
// 创建客户端存根来执行 RPC
client := pb.NewRouteGuideClient(conn)
// 调用服务方法 
// 1.简单远程调用（一元RPC）
feature, err := client.GetFeature(context.Background(), &pb.Point{409146138, -746188906})
if err != nil {
  ...
}
// 读取服务器的响应信息
log.Println(feature)

// 2.服务器端流式 RPC 
rect := &pb.Rectangle{ ... }  // initialize a pb.Rectangle
// 调用服务器端的流方法ListFeatures，它返回一个地理流Feature
stream, err := client.ListFeatures(context.Background(), rect)
if err != nil {
  ...
}
for {
    // 重复读取服务器对响应协议缓冲区对象的响应，直到没有更多消息为止
    feature, err := stream.Recv()
    // Recv()如果nil，则流仍然良好，可以继续阅读；如果是 io.EOF 那么消息流已经结束；否则一定是 RPC 错误，通过err
    if err == io.EOF {
        break
    }
    if err != nil {
        log.Fatalf("%v.ListFeatures(_) = _, %v", client, err)
    }
    log.Println(feature)
}

```

>   其他两种方式参考 https://grpc.io/docs/languages/go/basics/



## 拦截器
