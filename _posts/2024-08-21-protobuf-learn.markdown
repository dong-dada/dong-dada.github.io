---
layout: post
title:  "Protocol Buffer 分析"
date:   2024-08-21 10:57:19 +0800
categories: none
---

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

完整的编码格式参考 [Protobuf 官网](https://protobuf.dev/programming-guides/encoding/)，这里讨论它的一些特点：

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

就是说相同内容的 PB 对象编码后不一定会产生相同的数据，[这篇文章](https://protobuf.dev/programming-guides/serialization-not-canonical/) 解释了为什么设计成这样。

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

[这篇文章](https://protobuf.dev/overview/#not-good-fit) 提到 Protobuf 不适合存储超过 MB 级别的数据。在 C++ 中，PB 消息的 string, bytes 字段是通过 `std::string` 来存储的，因此编解码过程会产生拷贝开销。在 MB 级别数据量的情况下，拷贝开销可以达到 ms 级别，导致编解码性能大受影响。


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
