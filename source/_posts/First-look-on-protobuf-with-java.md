---
title: First look on protobuf with java
date: 2018-10-31 17:50:23
tags:
        - protobuf
        - java
        - code
---

# protobuf 性质速览

## proto2
protobuf语法分为proto2和proto3，此处只介绍proto2的语法。
### Message
protobuf数据以message为单元
```
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}
```
message每条filed由以下构成
rules type name = number;[option]
rules 用来限定该filed属性
* required
* optional
* repeated

<!--more-->

required表示一定需要该filed。
optional表示可选filed。可以设置optional的默认值，如果未设置，protobuf会依据filed的type设置默认的默认值。（实际上在parse过程中，protobuf会为optional filed设置一个默认值）
```
optional int32 result_per_page = 3 [default = 10];
```
repeated表示可重复filed。

type 描述该filed类型
* scalar type
* message
* enumeration

name表示该filed名字
number用来在message binary format中标识该filed

reversed filed
当你将旧有的filed删除或注释掉的之后，添加新的filed使用旧有number，这看起来似乎没有什么问题，但当你的程序读取旧数据则会产生data corruption等问题。为了避免这种情况，推荐将不再使用的filed设置成为reversed。
```
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```
### import
当你需要使用其他文件定义的message，先import该message所在的文件。
```
import "myproject/other_protos.proto";
```
默认情况下，你只能引用在该文件中定义的类型。如果你想使用，引用文件中引用的类型，请在引用文件中使用import public。这么说可能过于复杂，设想以下情景。当你决定移动某个proto文件，而不想逐一修改引用该proto文件的引用语句，可以这样操作。将proto文件移动到new\_proto文件，在old\_proto文件中使用import pulic来引用new\_proto。
你需要给protocol compiler添加**-I/--proto_path**参数来表示proto文件所在路径，默认情况下会是compiler被调用的路径。

### scalar value types
protobuf支持的原生type。[scalar types](https://developers.google.com/protocol-buffers/docs/proto#Scalar%20Value%20Types)
### Enumerations
类似于java中的enumeration，你可以将其定义在message中也可以定义在message外，如果你想使用其他message中的enum，使用这种写法MessageType.EnumType。注意：enum的数值常量应在32bit以内。由于enum使用变长编码，负数值会造成低效率故不推荐。
```
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3 [default = 10];
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  optional Corpus corpus = 4 [default = UNIVERSAL];
}
```
同样你也可以将enum的value设为保留值，以避免读取旧数据时产生问题。
```
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```
### nested types
除了在message内部定义enum，你也可以在message内部定义message。如果你想在外部使用该message，使用这种语法Parent.Type。
```
message SearchResponse {
  message Result {
    required string url = 1;
    optional string title = 2;
    repeated string snippets = 3;
  }
  repeated Result result = 1;
}
message SomeOtherMessage {
  optional SearchResponse.Result result = 1;
}
```
### Extensions
你可以在message内部声明一个一段数字作为extension保留。允许其他proto给该message添加新filed。
```
message Foo {
  // ...
  extensions 100 to 199;
}
extend Foo {
  optional int32 bar = 126;
}
```
虽然Foo编码后，其wire format同定义了一个新的filed的Foo相同，但是代码内访问该filed和访问普通filed有些不同。
```
Foo foo;
foo.SetExtension(bar, 15);
```
可以在message内部定义一个extension
```
message Baz {
  extend Foo {
    optional int32 bar = 126;
  }
  ...
}
```
它需要这样访问（c++code）
```
Foo foo;
foo.SetExtension(Baz::bar, 15);
```
当然有个有趣的操作就是，你可以在定义一个extension在其本身中。
```
message Baz {
  extend Foo {
    optional Baz foo_ext = 127;
  }
  ...
}
```
这或许有些难以理解，其实这和以下等效。
```
message Baz {
  ...
}

// This can even be in a different file.
extend Foo {
  optional Baz foo_baz_ext = 127;
}
```
### Oneof
如果你有一些optional filed，同时很明确这些filed同时最多只有一个能被设置。你可以使用**oneof**
```
message SampleMessage {
  oneof test_oneof {
     string name = 4;
     SubMessage sub_message = 9;
  }
}
```
注意，oneof不能使用rules词（optional、required和repeated）修饰。如果我想repeated怎么办？把oneof的某个filed设为包含repeated filed的message type。
当你尝试给oneof设置值时，会清除其前一个值。
在c++中请小心使用，因为你很可能持有旧有引用而引起问题。
```
SampleMessage message;
SubMessage* sub_message = message.mutable_sub_message();
message.set_name("name");      // Will delete sub_message
sub_message->set_...            // Crashes here
```

### Maps
map类型定义如下
```
map<key_type, value_type> map_field = N;
```
key\_type可以是任意integer或者string类型，所以除float point类型和bytes类型外的其他任何scalar类型均可以做为key\_type.
注意
* 不支持extension
* 不支持repeated，optional和required
* wire format 顺序和map迭代顺序并未定义，因此你不能依赖map item顺序
* 生成的text format中，maps会按key排序。
* 当解析wire format时，如果出现重复key，则会以最后一个为准，但是解析text format，重复key会出错。

兼容性，对于不支持map的protobuf实现，会议以下方式处理map。
```
message MapFieldEntry {
  optional key_type key = 1;
  optional value_type value = 2;
}

repeated MapFieldEntry map_field = N;
```

### Packages
使用package来避免.proto的命名冲突。
```
package foo.bar;
message Open { ... }
```
可以这样使用Open
```
message Foo {
  ...
  required foo.bar.Open open = 1;
  ...
}
```
### Services
如果你想在RPC中使用你的message类型，你可以在.proto文件中定义RPC service interface接口。随后protobuf compiler会自动根据你的语言生产服务接口代码和桩代码。比如你想创建一个使用SearchRequest和SearchResponse的RPC服务，你可以做出如下定义。
```
service SearchService {
  rpc Search (SearchRequest) returns (SearchResponse);
}
```
### Options
java常用option
* java\_package (file option) 定义生产代码所在包
* java\_outer\_classname (file option) 定义生成代码类名
* optimize\_for (file option)： 优化方式

## proto3
tbd

## Encoding
涉及底层编码方式
tbd

## protobuf for JAVA

简单定义一个person.proto
```
syntax = "proto2";

package tutorial;

option java_package = "com.example.tutorial";
option java_outer_classname = "AddressBookProtos";

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
生成java code，可以使用command-line
```
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/addressbook.proto
```
如果使用maven,maven可以自动帮你生成相应代码。
```
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>2.5.0</version>
</dependency>
...
<plugin>
    <groupId>org.xolstice.maven.plugins</groupId>
    <artifactId>protobuf-maven-plugin</artifactId>
    <version>0.5.0</version>
    <configuration>
        <protocExecutable>protoc</protocExecutable>
        <protoSourceRoot>${basedir}/protobuf</protoSourceRoot>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>compile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
```java
// Serializer code
Person john =
        Person.newBuilder()
                .setId(1234)
                .setName("John Doe")
                .setEmail("jdoe@example.com")
                .addPhones(Person.PhoneNumber.newBuilder()
                                .setNumber("555-4321")
                                .setType(Person.PhoneType.HOME))
                .build();

Person kulisu =
        Person.newBuilder()
                .setId(1000)
                .setName("kulisu")
                .setEmail("Kulisu@outlook.com")
                .addPhones(Person.PhoneNumber.newBuilder()
                        .setNumber("173xxxx8086")
                        .setType(Person.PhoneType.MOBILE))
                .build();
AddressBook addressBook =
        AddressBook.newBuilder()
                    .addPeople(john)
                    .addPeople(kulisu)
                    .build();
File file = new File("person.data");
try (FileOutputStream fileOutputStream = new FileOutputStream(file)) {
    addressBook.writeTo(fileOutputStream);

} catch (IOException e) {
    e.printStackTrace();
}

// parser code

File file = new File("person.data");
try (FileInputStream fileInputStream = new FileInputStream(file)) {
    AddressBook addressBook = AddressBook.parseFrom(fileInputStream);
    for (Person person: addressBook.getPeopleList()) {
        System.out.println("Person ID: " + person.getId());
        System.out.println("  Name: " + person.getName());
        if (person.hasEmail()) {
            System.out.println("  E-mail address: " + person.getEmail());
        }

        for (Person.PhoneNumber phoneNumber : person.getPhonesList()) {
            switch (phoneNumber.getType()) {
                case MOBILE:
                    System.out.print("  Mobile phone #: ");
                    break;
                case HOME:
                    System.out.print("  Home phone #: ");
                    break;
                case WORK:
                    System.out.print("  Work phone #: ");
                    break;
            }
            System.out.println(phoneNumber.getNumber());
        }
    }
} catch (FileNotFoundException e) {
    e.printStackTrace();
} catch (IOException e) {
    e.printStackTrace();
}
```
执行结果如下
```
Person ID: 1234
  Name: John Doe
  E-mail address: jdoe@example.com
  Home phone #: 555-4321
Person ID: 1000
  Name: Kulisu
  E-mail address: Kulisu@outlook.com
  Mobile phone #: 173xxxx8086
```
