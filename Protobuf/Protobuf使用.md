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



### 标识号

每一个字段都有唯一的标识号，该标识号是用于消息的二进制定义，一旦编译，则不应该再改动标识号。1-15标识号的字段使用一个字节编码，而16-2047则使用两个字节编码。因此常用的字段应使用1-15来标识，并且应预留标识号给日后使用。



### 修饰符

2.0 语法中字段必须定义修饰符：

- `required`: 该字段必须提供值，否则该消息会被定义为未初始化。如果尝试序列化未初始化的对象则会抛 `RuntimeException`； 反序列化一个未初始化的消息会抛 `IOException`。除此之外和 optional 一样
- `optional`: 该字段可被设值。如果没设值，会设值为该字段类型的默认值（可以通过 `[default = xxx]` 自定义默认值）。
- `repeated`: 可以看做为数组，设入的值会依次保留在 protocol buffer 中。（java 中会将该字段编译为 List 集合）



3.0 语法中可以忽略不定义修饰符，并且默认使用 repeated。删除了 `required` 和 `optional`

* `singular` :单一字段。该字段类似于 `optional` ，只能为0个或1个。（会被设置默认值）



默认值：

- Strings 类型默认值为空字符串。
- Bytes 默认值为空的 Bytes。
- Bools 默认值为 false。
- 数值型默认为0。
- 枚举类型默认值为第一个枚举值（标识号为0）。



### 枚举类

```protobuf
enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
}
```

枚举类标识号从 0 开始。



### 注释

使用 `//` 或`/* ... */` 。



### 字段保留

当通过删除字段来更新 Message 时，可能会出现复用删除的标识号。通过保留标识号或字段名，来防止调用已经删除的字段或标识号（如果使用则会报错）。

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

注：不能在同一行 `reserved` 中同时修饰标识号和字段名。



### Message 类型

字段可以定义其他 Message 类型。该Message 须定义到同一个 `.proto` 文件中。如果不在同一个文件中则可以通过 `import "myproject/other_protos.proto";` 引入该 `.proto` 文件。



### 更新 Message 类型

如果一个已有的消息格式已无法满足新的需求——如，要在消息中添加一个额外的字段——但是同时旧版本写的代码仍然可用。不用担心！更新消息而不破坏已有代码是非常简单的。在更新时只要记住以下的规则即可。

- 不要更改任何已有的字段的数值标识。
- 如果你增加新的字段，使用旧格式的字段仍然可以被你新产生的代码所解析。你应该记住这些元素的默认值这样你的新代码就可以以适当的方式和旧代码产生的数据交互。相似的，通过新代码产生的消息也可以被旧代码解析：只不过新的字段会被忽视掉。注意，未被识别的字段会在反序列化的过程中丢弃掉，所以如果消息再被传递给新的代码，新的字段依然是不可用的（这和proto2中的行为是不同的，在proto2中未定义的域依然会随着消息被序列化）
- 非required的字段可以移除——只要它们的标识号在新的消息类型中不再使用（更好的做法可能是重命名那个字段，例如在字段前添加“OBSOLETE_”前缀，那样的话，使用的.proto文件的用户将来就不会无意中重新使用了那些不该使用的标识号）。
- int32, uint32, int64, uint64,和bool是全部兼容的，这意味着可以将这些类型中的一个转换为另外一个，而不会破坏向前、 向后的兼容性。如果解析出来的数字与对应的类型不相符，那么结果就像在C++中对它进行了强制类型转换一样（例如，如果把一个64位数字当作int32来 读取，那么它就会被截断为32位的数字）。
- sint32和sint64是互相兼容的，但是它们与其他整数类型不兼容。
- string和bytes是兼容的——只要bytes是有效的UTF-8编码。
- 嵌套消息与bytes是兼容的——只要bytes包含该消息的一个编码过的版本。
- fixed32与sfixed32是兼容的，fixed64与sfixed64是兼容的。
- 枚举类型与int32，uint32，int64和uint64相兼容（注意如果值不相兼容则会被截断），然而在客户端反序列化之后他们可能会有不同的处理方式，例如，未识别的proto3枚举类型会被保留在消息中，但是他的表示方式会依照语言而定。int类型的字段总会保留他们的



### Any

Any类型消息允许你在没有指定他们的.proto定义的情况下使用消息作为一个嵌套类型。一个Any类型包括一个可以被序列化bytes类型的任意消息，以及一个URL作为一个全局标识符和解析消息类型。为了使用Any类型，你需要导入`import google/protobuf/any.proto`

```protobuf
import "google/protobuf/any.proto";

message ErrorStatus {
  string message = 1;
  repeated google.protobuf.Any details = 2;
}
```

对于给定的消息类型的默认类型URL是`type.googleapis.com/packagename.messagename`。

不同语言的实现会支持动态库以线程安全的方式去帮助封装或者解封装Any值。例如在java中，Any类型会有特殊的`pack()`和`unpack()`访问器，在C++中会有`PackFrom()`和`UnpackTo()`方法。



### Oneof

如果你的消息中有很多可选字段，并且同时至多一个字段会被设置，你可以加强这个行为，使用oneof特性节省内存。

Oneof字段就像可选字段，除了它们会共享内存，至多一个字段会被设置。设置其中一个字段会清除其它字段。 你可以使用`case()`或者`WhichOneof()` 方法检查哪个oneof字段被设置，看你使用什么语言了。

为了在.proto定义Oneof字段， 你需要在名字前面加上oneof关键字, 比如下面例子的test_oneof:

```protobuf
message SampleMessage {
  oneof test_oneof {
    string name = 4;
    SubMessage sub_message = 9;
  }
}
```

然后你可以增加oneof字段到 oneof 定义中。你可以增加任意类型的字段，但是不能使用repeated 关键字。

在产生的代码中，oneof字段拥有同样的 getters 和setters，就像正常的可选字段一样。也有一个特殊的方法来检查到底那个字段被设置。

### Oneof 特性

- 设置oneof会自动清楚其它oneof字段的值。所以设置多次后，只有最后一次设置的字段有值。

- 如果解析器遇到同一个oneof中有多个成员，只有最会一个会被解析成消息。
- oneof不支持`repeated`。



### ProtoBuf 字段对应Java类型

| .proto Type | Notes                                                        | Java Type  |
| ----------- | ------------------------------------------------------------ | ---------- |
| double      |                                                              | double     |
| float       |                                                              | float      |
| int32       | 使用变长编码，对于负值的效率很低，如果你的域有可能有负值，请使用sint64替代 | int        |
| uint32      | 使用变长编码                                                 | int        |
| uint64      | 使用变长编码                                                 | long       |
| sint32      | 使用变长编码，这些编码在负值时比int32高效的多                | int        |
| sint64      | 使用变长编码，有符号的整型值。编码时比通常的int64高效。      | long       |
| fixed32     | 总是4个字节，如果数值总是比总是比228大的话，这个类型会比uint32高效。 | int        |
| fixed64     | 总是8个字节，如果数值总是比总是比256大的话，这个类型会比uint64高效。 | long       |
| sfixed32    | 总是4个字节                                                  | int        |
| sfixed64    | 总是8个字节                                                  | long       |
| bool        |                                                              | boolean    |
| string      | 一个字符串必须是UTF-8编码或者7-bit ASCII编码的文本。         | String     |
| bytes       | 可能包含任意顺序的字节数据。                                 | ByteString |



## ProtoBuf API

1.  `.proto` 文件中定义的消息定义成类（final 类，如果需要改动则应重新编译）内嵌在生成的 `.java` 文件中，并提供该类的 Builder 类来实例化该类。
2. Message 类中提供了 `getter()` 方法来获取值，而 Builder 类提供了 `setter()` 和 `getter()` 方法。
3. 每个 Builder 类提供 `clear()` 方法清空值。
4. repeated 修饰的字段，会编译为 List 集合，因此在 get 、set 中提供 index 参数来获取指定位置的元素/插入到指定位置。

5. 通过 `Message.newBuilder()` 创建 Builder 类，并通过 `setter()` 对类的字段设值，最后通过 `Builder.build()` 创建实例。
6. `isInitialized()`: 检查是否所有 required 修饰的字段都设值了。
7. `toString()`: 将字段输出，一般用于 debug。
8. `mergeFrom(Message other)`: (builder only) 将 `other` 合并到该 message 中，覆盖掉单一字段，对于 repeated 字段则添加进 List 中。
9. `clear()`: (builder only) 清空所有字段值为默认值。



注：如果不确定 `.proto` 文件的定义或希望有更多的方法，则应该使用装饰类装饰 Message，而不是继承 Message 类或频繁修改 `.proto` 文件。



## 扩展 ProtoBuf

当你扩展 ProtoBuf 时，为了向后兼容，必须遵循以下几点：

- 不能更改已有字段的标识号。

- 不能添加或删除 required 的字段。

- 可以删除 optional、repeated 字段。

- 可以添加 optional、repeated 字段，但必须使用新的标识号（不能是被删除字段的标识号）。


## 序列化和反序列化

- `byte[] toByteArray();`: 序列化为二进制数组。
- `static Person parseFrom(byte[] data);`: 将二进制数组反序列化。
- `void writeTo(OutputStream output);`: 序列化并写到 `OutputStream`。
- `static Person parseFrom(InputStream input);`: 从 `InputStream`反序列化。







官方资料：https://developers.google.com/protocol-buffers/docs/proto3#updating