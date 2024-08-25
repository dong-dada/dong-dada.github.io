---
layout: post
title:  "Protocol Buffer 学习"
date:   2024-08-21 10:57:19 +0800
---

* TOC
{:toc}

本文从各种角度讨论 Protobuf 值得关注的点。

## 编码

### Protoscope 工具

[Protoscope](https://github.com/protocolbuffers/protoscope) 工具可以把编码后的 PB 数据用人类可读的方式展示出来：

```
// 定义 proto
syntax = "proto3";

message Person {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

// 创建 PB 对象并序列化到 person.pb 文件
int main() {
  Person person;
  person.set_id(42);
  person.set_name("alice");
  person.set_email("hello@outlook.com");

  std::cout << person.SerializeAsString();

  return 0;
}

// 使用 protoscope 工具查看
$ protoscope -explicit-wire-types -explicit-length-prefixes person.pb
1:VARINT 42
2:LEN 5 "alice"
3:LEN 17 "hello@outlook.com"
```

### Proto2 和 Proto3 的编码格式是相同的

Proto2 和 Proto3 在语法上有差异，但在编码格式上是相同的，这意味着可以平滑地把语法从 Proto2 升级到 Proto3。


### Protobuf 编码不具备自解释性

完整的编码格式参考 [这篇文章](https://protobuf.dev/programming-guides/encoding/)，这里讨论它的一些特点：

```
message    := (tag value)*

tag        := (field << 3) bit-or wire_type;
                encoded as uint32 varint
value      := varint      for wire_type == VARINT,
              i32         for wire_type == I32,
              i64         for wire_type == I64,
              len-prefix  for wire_type == LEN,
              <empty>     for wire_type == SGROUP or EGROUP
```

可以看出这里的编码思路是将 message 的每个字段编码为 `wire_type-field_number-value` 的形式。其中 `wire_type` 指明了 value 采用何种编码方式，`field_number` 就是 message 中字段的序号, `value` 是具体的值。

值得注意的是 `wire_type` 并不是跟具体的类型相关的，比如 I32 既可以是 fixed32 也可以是 float，LEN 既可以表示 string 也可以表示内嵌消息，这意味着 Protobuf 的编码格式不具备“自解释性”。

这一点可以从 protoscope 工具看出：

```
// 定义 proto, 注意内嵌消息
syntax = "proto3";

message Person {
  int32 id = 1;
  string name = 2;
  string email = 3;

  message PhoneNumber {
    string number = 1;
    string code = 2;
  }

  PhoneNumber phone = 4;
}

// 创建 PB 对象并序列化
int main() {
  Person person;
  person.set_id(42);
  person.set_name("alice");
  person.set_email("hello@outlook.com");
  person.mutable_phone()->set_number("12345678900");
  person.mutable_phone()->set_code("86");

  std::cout << person.SerializeAsString();
}

// 使用 protoscope 工具查看，它没能正确识别内嵌消息
$ protoscope -explicit-wire-types -explicit-length-prefixes person.pb
1:VARINT 42
2:LEN 5 "alice"
3:LEN 17 "hello@outlook.com"
4:LEN 17
  1:LEN 11
    6:I64 4.663768570166562e-33   # 0x3938373635343332i64
    6:VARINT 48
  2:LEN 2 7:VARINT 54
```

不具备自解释性，使得我们无法编写直接将 PB 数据转换为 json 或者 map 之类的转换器，在处理 PB 数据时必须有 proto 文件才可以。比如 [pbjson](https://github.com/yinqiwen/pbjson/blob/master/test/test.cpp) 这个 C++ 库，它提供的 pb2json 方法需要传入 PB message 对象才可以做解析。

### Protobuf 编码不是稳定的

就是说相同内容的 PB 对象编码后不一定会产生相同的数据，会有顺序的差别，[这篇文章](https://protobuf.dev/programming-guides/serialization-not-canonical/) 解释了为什么设计成这样。

由于编码不稳定，在做比较的时候不能直接比较编码后数据，只能解码之后通过 PB 对象进行比较：

```cpp
#include <google/protobuf/util/message_differencer.h>

using google::protobuf::util::MessageDifferencer;

bool pb_equal(std::string_view lhs, std::string_view rhs) {
    // 直接比较是不行的
    // return lhs == rhs;

    auto lhs_obj = Person::ParseFromString(lhs);
    auto rhs_obj = Person::ParseFromString(rhs);

    // 使用 DebugString 进行比较也是不行的
    // return lhs_obj.DebugString() == rhs_obj.DebugString();

    // 应当使用 MessageDifferencer 工具类进行比较
    // MessageDifferencer 还提供了 Equivalent(), 
    // ApproximatelyEquals(), ApproximatelyEquivalent() 等方法，
    // 适合不同场景，使用的时候再看看
    return MessageDifferencer::Equals(alice, bob);
}
```

显然由于不稳定性无法直接从编码后的 PB 数据计算 hash，对于解码后的 PB message 对象，Protobuf 也没有提供计算 hash 的方法。我理解针对 message 对象做 hash 是可以办到的，只是需要明确规定对于 unknown field, default value 的处理方式。

### Protobuf 编码没有定界符

[这篇文章](https://protobuf.dev/programming-guides/techniques/#streaming) 介绍了这件事。

因此流式传输的时候需要添加额外的长度字段来标记一份 PB 数据的开始和结束。

这也意味着 PB 解码的时候，传入不完整的数据或者超长的数据，解码不一定会失败。我们不能根据解码是否成功来判断 PB 数据的完整性，必须依靠额外的定界符来判断。

### Protobuf 编码的尺寸限制

[Encoding](https://protobuf.dev/programming-guides/encoding/) 这篇文章中说明了编码时的尺寸限制：
- string, bytes 类型的字段，单个字段不能超过 2GB
- message 整体不能超过 2GB

之所以是 2GB 是因为 LEN 类型的 wire_type 使用 int32 类型的 VARINT 来存储长度信息。

[这篇文章](https://protobuf.dev/overview/#not-good-fit) 提到 Protobuf 不适合存储超过 MB 级别的数据。在 C++ 中，PB 消息的 string, bytes 字段是通过 `std::string` 来存储的，因此编解码过程会产生拷贝开销。在 MB 级别数据量的情况下，拷贝开销可以达到 ms 级别，导致编解码性能大受影响。

[Protobuf 的讨论组](https://groups.google.com/g/protobuf/c/eNQ02xdhOAE) 提到让 string 类型支持 `std::string_view` 的事情，用来避免拷贝开销，目前还没有支持。


## 用法

### google.protobuf.Any

`google.protobuf.Any` 可以存储任意类型的消息，使用方法:

```proto
import "google/protobuf/any.proto";

message Error {
  string message = 1;
  google.protobuf.Any detail = 2;
}
```

```cpp
// 把任意消息编码到 any 字段当中
NetworkErrorDetail detail = ...;
Error error;
status.mutable_detail()->PackFrom(detail);

// ...

// 从 any 字段中解码出原始消息
if (status.detail().Is<NetworkErrorDetail>()) {
    NetworkErrorDetail detail;
    status.detail().UnpackTo(&detail);
}
```

从 `google.protobuf.Any` 消息的定义可以很容易地理解它的工作原理，它只是把任意消息编码为 bytes 存储而已：

```proto
package google.protobuf;

message Any {
  string type_url = 1;      // 原始消息的类型
  bytes value = 2;          // 编码后的消息
}
```

[这篇文章](https://protobuf.dev/programming-guides/techniques/#self-description) 提到了使用 Any 和 FileDescriptorSet 实现自解释消息的思路。虽然 Protobuf 编码格式自身不具备自解释性，但是可以把描述信息(也就是 .proto 文件定义的内容)也编码到数据内，使得编码后的数据能够通过反射的方式进行访问。

### google.protobuf.Struct

`google.protobuf.Struct` 可以实现像 json 那样的效果，它的定义就是一个 map:

```proto
message Struct {
  map<string, Value> fields = 1;
}

message Value {
  oneof kind {
    NullValue null_value = 1;
    double number_value = 2;
    string string_value = 3;
    bool bool_value = 4;
    Struct struct_value = 5;
    ListValue list_value = 6;
  }
}
```

### Options

[这篇文章](https://protobuf.dev/programming-guides/proto3/#options) 提到 Protobuf 支持在 proto 文件中定义选项:

```proto
syntax = "proto3";

// 文件级别的 option
option c_enable_arenas = true;

message Person {
  // 消息级别的 option
  option deprecated = true;

  int32 id = 1;
  string name = 2 [deprecated = true]; // 字段级别的 option
  string email = 3;
}
```

值得注意的是 [protovalidate](https://github.com/bufbuild/protovalidate) 这个项目，使用它可以在 proto 文件中加入一些校验用的 option，protoc 在生成目标代码的时候会产生额外的校验信息，可以验证消息是否满足要求：

```proto
syntax = "proto3";

import "buf/validate/validate.proto";

message Transaction {
  // 要求 id 值大于 999
  uint64 id = 1 [(buf.validate.field).uint64.gt = 999];
}
```

```cpp
#include <iostream>

#include "buf/validate/validator.h"
#include "path/to/generated/protos/transaction.pb.h"

int main() {
  my::package::Transaction transaction;
  transaction.set_id(1234);

  auto factory = buf::validate::ValidatorFactory::New().value();
  auto validator = factory->NewValidator();

  // 校验
  auto results = validator.Validate(transaction).value();
  if (results.violations_size() > 0) {
    std::cout << "validation failed" << std::endl;
  } else {
    std::cout << "validation succeeded" << std::endl;
  }
  return 0;
}
```

### Reflection

通过反射可以在不知道具体类型的情况下访问 PB 对象，比如 pb 转 json 之类的场景就很适合用这种方式:

```cpp
JsonValue pb2json(const Message& message) {
  // Descriptor 描述了消息的静态信息(有哪些字段，每个字段什么类型)
  // Reflection 记录了消息的动态信息(字段的具体值是什么)
  auto descriptor = message.GetDescriptor();
  auto reflection = message.GetReflection();

  JsonMap json;
  for (int i = 0; i < descriptor->field_count(); ++i) {
    auto field = descriptor->field(i);

    auto key = field->name();

    switch (field->cpp_type()) {
      case FieldDescriptor::CPPTYPE_BOOL:
        json.addBool(key, reflection->GetBool(message, field));
        break;
      case FieldDescriptor::CPPTYPE_INT32:
        json.addInt(key, reflection->GetInt32(message, field));
        break;
      case FieldDescriptor::CPPTYPE_MESSAGE:
        json.addValue(key, pb2json(reflection->GetMessage(message, field)));
        break;
        // ...
    }
  }

  return json;
}
```


## 源码

### 数据的存储

从 .proto 文件生成的 PB 对象会有相应字段来存储数据：

```proto
message Person {
  int32 id = 1;
  string name = 2;
  string email = 3;

  message PhoneNumber
  {
    string zip_code = 1;
    string number = 2;
  }

  PhoneNumber phone = 4;
}
```

```cpp
class Person_PhoneNumber final : public ::PROTOBUF_NAMESPACE_ID::Message {
private:
  internal::ArenaStringPtr zip_code_;
  internal::ArenaStringPtr number_;
};

class Person final : public ::PROTOBUF_NAMESPACE_ID::Message {
private:
  int32 id_;
  internal::ArenaStringPtr name_;
  internal::ArenaStringPtr email_;
  ::Person_PhoneNumber* phone_;
};
```

其中 ArenaStringPtr 跟 [Arena 机制](https://protobuf.dev/reference/cpp/arenas/) 有关，没开启 arena 的话可以认为它是一个 `std::string` 指针。

### 编解码

生成的 PB 对象中有相应的编解码方法，当调用父类 `google::protobuf::MessageLite` 的编解码方法时，会触发 PB 对象的编解码方法:

```cpp
class Person final : public ::PROTOBUF_NAMESPACE_ID::Message {
public:
  size_t ByteSizeLong() const final;
  const char* _InternalParse(const char* ptr, internal::ParseContext* ctx) final;
  uint8* _InternalSerialize(uint8* target, io::EpsCopyOutputStream* stream) const final;
}
```

```cpp
uint8* Person::_InternalSerialize(uint8* target, io::EpsCopyOutputStream* stream) const {
  if (this->id() != 0) {
    target = stream->EnsureSpace(target);
    target = internal::WireFormatLite::WriteInt32ToArray(1, this->_internal_id(), target);
  }

  if (!this->name().empty()) {
    target = stream->WriteStringMaybeAliased(2, this->_internal_name(), target);
  }

  if (!this->email().empty()) {
    target = stream->WriteStringMaybeAliased(3, this->_internal_email(), target);
  }

  // ...
}
```

```cpp
const char* Person::_InternalParse(const char* ptr, internal::ParseContext* ctx) {
  while (!ctx->Done(&ptr)) {
    uint32 tag;
    ptr = internal::ReadTag(ptr, &tag);

    // tag >> 3 得到 field number
    // 根据 field number 执行不同的解析方法
    switch (tag >> 3) {
      case 1:
        id_ = internal::ReadVarint64(&ptr);
        break;
      case 2:
        auto name = _internal_mutable_name();
        ptr = ::PROTOBUF_NAMESPACE_ID::internal::InlineGreedyStringParser(name, ptr, ctx);
        break;
      case 3:
        auto email = _internal_mutable_email();
        ptr = ::PROTOBUF_NAMESPACE_ID::internal::InlineGreedyStringParser(email, ptr, ctx);
        break;
      // ...
    }
  }
}

size_t Person::ByteSizeLong() const {
  size_t total_size = 0;

  if (this->id() != 0) {
    total_size += 1 + internal::WireFormatLite::Int32Size(this->_internal_id());
  }

  if (!this->name().empty()) {
    total_size += 1 + internal::WireFormatLite::StringSize(this->_internal_name());
  }

  if (!this->email().empty()) {
    total_size += 1 + ::internal::WireFormatLite::StringSize(this->_internal_email());
  }

  // ...

  int cached_size = ::internal::ToCachedSize(total_size);
  SetCachedSize(cached_size);
  return total_size;
}
```

值得注意的是 `ByteSizeLong()` 方法每次执行都会遍历所有字段，在延迟敏感场景下它对性能带来的影响不可忽视。可以看到上述代码中最后一段执行了 `SetCachedSize()` 方法来缓存总大小，这个缓存起来的值需要通过 `GetCachedSize()` 获取，在需要时可以调用该方法，但是要注意保证消息没有被改动过。


### Serialize 接口

PB 对象的基类 `google::protobuf::MessageLite` 提供了许多序列化用的方法:

```cpp
class MessageLite {
  bool SerializeToCodedStream(io::CodedOutputStream* output) const;
  bool SerializeToZeroCopyStream(io::ZeroCopyOutputStream* output) const;
  bool SerializeToString(std::string* output) const;
  bool SerializeToArray(void* data, int size) const;
  std::string SerializeAsString() const;
  bool SerializeToFileDescriptor(int file_descriptor) const;
  bool SerializeToOstream(std::ostream* output) const;
  bool AppendToString(std::string* output) const;

  // SerializePartial* 系列，与上述方法对应，只是不会检查 required 字段是否被设置
  bool SerializePartialToCodedStream(io::CodedOutputStream* output) const;
  bool SerializePartialToZeroCopyStream(io::ZeroCopyOutputStream* output) const;
  bool SerializePartialToString(std::string* output) const;
  bool SerializePartialToArray(void* data, int size) const;
  std::string SerializePartialAsString() const;
  bool SerializePartialToFileDescriptor(int file_descriptor) const;
  bool SerializePartialToOstream(std::ostream* output) const;
  bool AppendPartialToString(std::string* output) const;
}
```

这些序列化方法的输出各有不同，但最终都会转换为 `io::EpsCopyOutputStream` 传入给 PB 对象的 `_InternalSerialize()` 方法。

```mermaid
classDiagram

class ZeroCopyOutputStream {
    <<interface>>
}

class ArrayOutputStream
class OstreamOutputStream
class FileOutputStream
class EpsCopyOutputStream

ZeroCopyOutputStream <|-- ArrayOutputStream
ZeroCopyOutputStream <|-- OstreamOutputStream
ZeroCopyOutputStream <|-- FileOutputStream
EpsCopyOutputStream --> ZeroCopyOutputStream
```

上面类图说明了 OutputStream 之间的关系，有若干具体类实现了`ZeroCopyOutputStream` 接口，这些具体类分别用于序列化到数组或字符串、`std::ostream`、文件描述符。

`ZeroCopyOutputStream` 接口定义了如何申请和退回内存的逻辑，`EpsCopyOutputStream` 调用它的方法申请内存，随后写入各种数据。

```cpp
class ZeroCopyOutputStream {
public:
  // 获取一个可写的内存块，返回这个内存块的地址和大小
  // 如果内存块不够大，会再次调用 Next() 去申请
  virtual bool Next(void** data, int* size) = 0;

  // 序列化结束了，退回多余的内存块
  virtual void BackUp(int count) = 0;

  // 返回目前已经分配的内存总大小
  virtual int64_t ByteCount() const = 0;
};
```

值得注意的是用户可以自己实现 `ZeroCopyOutputStream` 接口，随后调用 `SerializeToZeroCopyStream()` 进行序列化，从而自定义内存分配过程。比如可以每次 `Next()` 被调用都分配一块新的内存，最后把 PB 消息序列化到多块内存上。


### Parse 接口

PB 对象的基类 `google::protobuf::MessageLite` 提供了许多反序列化用的方法:

```cpp
class MessageLite {
  bool ParseFromCodedStream(io::CodedInputStream* input);
  bool ParseFromZeroCopyStream(io::ZeroCopyInputStream* input);
  bool ParseFromFileDescriptor(int file_descriptor);
  bool ParseFromIstream(std::istream* input);
  bool ParseFromString(ConstStringParam data);
  bool ParseFromArray(const void* data, int size);

  // ParsePartialFrom* 系列方法，未列出
}
```

类似的，可以自行实现 `ZeroCopyInputStream` 接口：

```cpp
class ZeroCopyInputStream {
public:
  // 获取一块可供读取的内存
  virtual bool Next(const void** data, int* size) = 0;

  // 上层读取到某个阶段时，发现剩余数据不足以完整读取了，就会退回剩余数据。
  // 下次调用 Next() 时需要返回更多数据
  virtual void BackUp(int count) = 0;

  // 跳过一部分数据
  virtual bool Skip(int count) = 0;

  // 已经提供的内存总大小
  virtual int64_t ByteCount() const = 0;
};
```
