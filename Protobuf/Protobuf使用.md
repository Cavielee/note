## 什么是 Protobuf

　　ProtoBuf 全程 Google Protocol Buffers，它由谷歌开源而来。它将数据结构以 `.proto` 文件进行描述，通过代码生成工具可以生成对应数据结构的 POJO 对象和 Protobuf 相关的方法和属性。

特点：

* 使用二进制编码
* 支持多语言，平台无关
* 只需要注重定义数据描述文件（纯文本，语言无关性），即可根据所需的语言，进行自动代码生成。



## proto 文件定义

以下基于 3.6.1 版本。

例子：

```protobuf
syntax = "proto3";

package com.cavie.timeserver.netty.proto;

option java_package = "com.cavie.timeserver.netty.proto";
option java_outer_classname = "Time";

message Person {
  required string name = 1;
  required int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```



### 版本号

```protobuf
syntax = "proto3";
```

指定版本号，如果不指定，则默认使用2.0的语法。



### Package 定义

```protobuf
package com.cavie.timeserver.netty.proto;

option java_package = "com.cavie.timeserver.netty.proto";
```

定义生成的 Java 文件的包名。若同时定义，则以 java_package 为准。



### Java 文件名

```protobuf
option java_outer_classname = "Time";
```

定义编译后输出的 Java 文件名。若不定义则默认为 .proto文件名，若文件名为 "my_proto.proto" 则会以驼峰命名为"MyProto" 。



### 消息定义

```protobuf
message AddressBook {
  repeated Person people = 1;
}
```

消息实际是一组 `修饰符+字段类型+字段名+标识号`。



### 修饰符

2.0 语法中字段必须定义修饰符，3.0 中可以省略。

- `required`: 该字段必须提供值，否则该消息会被定义为未初始化。如果尝试序列化未初始化的对象则会抛 `RuntimeException`； 反序列化一个未初始化的消息会抛 `IOException`。除此之外和 optional 一样
- `optional`: 该字段可被设值。如果没设值，会设值为该字段类型的默认值（可以通过 `[default = xxx]` 自定义默认值）。字段默认值：数值则为0，字符串为空字符串，布尔为false。对于消息中嵌入消息，则默认值为消息默认实例/原型。
- `repeated`: 可以看做为数组，设入的值会依次保留在 protocol buffer 中。





https://developers.google.com/protocol-buffers/docs/javatutorial#defining-your-protocol-format



https://blog.csdn.net/fangxiaoji/article/details/78826165



https://blog.csdn.net/u011518120/article/details/54604615



