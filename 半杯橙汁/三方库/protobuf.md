> 本质和自己工作中写的TransObj一样，但是更完善。底层也是内存操作二进制数据。

## 安装

```bash
# 安装依赖项
sudo apt-get install autoconf automake libtool curl make g++ unzip
# 解压安装
tar -xzvf protobuf-3.20.1.tar.gz
cd protobuf-3.20.1/
./autogen.sh
./configure
make -j8
sudo make install
sudo ldconfig
```

# 使用protobuf

+ 写IDL(Interface Description Language)接口描述语言，即设计通信结构体

  ```protobuf
  //proto2格式
  message ReqSignup{
      //required 有且仅有一次 optional 0/1次 repeated 0/n次（可以用来构造数组）
      required string username = 1;  
      required string password = 2;
  }
  //proto3格式，仅删除了required
  syntax = "proto3";
  message ReqSignup{
      string username = 1;  
      string password = 2;
  }
  ```

+ 使用protoc编译器生成c++文件

  ```bash
  # 使用gRPC生成服务器和客户端文件
  protoc -I ../../protos --grpc_out=. --plugin=protoc-gen-grpc=`which grpc_cpp_plugin` ../../protos/route_guide.proto
  # protoc --proto_path(proto文件地址) --cpp_out(输出为cpp文件的地址) test.proto(源proto文件名)
  protoc --proto_path=. --cpp_out=. test.proto 
  ```

+  通过API进行使用

  ```c++
  #include <string>
  #include <iostream>
  #include <stdio.h>
  #include "test.pb.h"
  
  int main(){
      //创建消息体对象
      ReqSignup message;
      std::string username{"admin"};
      std::string password{"123"};
      message.set_username(username);
      message.set_password(password);
      //序列化
      std::string output;
      message.SerializeToString(&output);
      printf("output string :%s\n",output.c_str());
  }
  ```

  ```bash
  g++ test.cc test.pb.cc -lprotobuf -lpthread # 记得连接pthread，否则程序会出现std::system_error错误
  ```

  protobuf常用API：

  ```c++
  bool SerializeToString(string* output) const; //将消息序列化并储存在指定的string中。注意里面的内容是二进制的，而不是文本；我们只是使用string作为一个很方便的容器。
  
  bool ParseFromString(const string& data); //从给定的string解析消息。
  
  bool SerializeToArray(void * data, int size) const        //将消息序列化至数组
  
  bool ParseFromArray(const void * data, int size)        //从数组解析消息
  
  bool SerializeToOstream(ostream* output) const; //将消息写入到给定的C++ ostream中。
  
  bool ParseFromIstream(istream* input); //从给定的C++ istream解析消息。
  
  //带有repeated字段的消息，通过add_依次赋值。
  //使用mutable_，赋值时候，可以使用局部变量，因为在调用的时，内部做了new操作。
  //参考：https://blog.csdn.net/weixin_43795921/article/details/115474254
  ```

# proto3

## 使用指南

```protobuf
syntax = "proto3";
 
message SearchRequest {
    string query = 1;
    int32 page_number = 2;
    int32 result_per_page = 3;
}
```

定义一个IDL通常为以下步骤：

### 声明proto版本

protoc编译器默认使用`proto2`，如果想要使用`proto3`，必须在IDL文件的**第一行**(尤其注意vscode注释插件，可能自动插入注释)指明`syntax = "proto3";`

### 定义IDL

`message`关键字可以看做C++中的`struct`，该IDL定义了3个字段，对应C++中分别为：`std::string`，`int32`，`int32`。

### 分配字段序号

每个字段有**唯一**的序号。这些序号在序列化数据中标识对应的字段，一旦程序中使用了该字段，不应该再改变其序号。

> 注意，1-15范围内的序号仅占一个字节，之后会占两个字节。为了效率，任何IDL都中的序号都应该保持在1-15范围内

### 表明字段规则

+ Singular:
  - `optional`：（推荐），该字段是可选的，如果没有使用，则会设置为默认值，并且不会进行序列化；否则会根据设定值进行序列化。为了和`proto2`保持兼容推荐使用`optional`，而非`implicit`。
  - `implicit`:（不推荐）如果字段是一个message类型，同`optional`一样。否则，若字段显式设置为非零默认值或者根据反序列化得来的非零默认值。该字段将会进行序列化；若该字段设置为默认零值，将不会进行序列化。对于默认零值是设置的还是反序列化得来又或者根本没有提供，用户无法判断。
+ `repeated`:该字段可以重复0-n次。重复值的顺序也会被保留。
+ `map`:键值对类型

> `proto3`中，`repeated`字段的标量数字默认使用`packed`编码

### 定义多个消息类型

单个IDL文件中可以定义多个`message`

### 添加注释

注释风格同C++，可以使用`\\`或`\**\`进行注释 

### 保留而非直接删除字段

如果一个字段不再使用，在代码中已经完全剔除，不应该直接在IDL中删除，而应该使用`reserved `进行保留。这样做的好处是，在其他开发者使用的时候，不会因为使用了已经删除的字段而导致序列化反序列化不统一的问题。使用了`reserved`字段后，编译器会提供警告。

```protobuf
message Foo {
  reserved 2, 15, 9 to 11; //保留字段号 2,15,9,10,11
  reserved "foo", "bar";	//保留字段名 foo bar
}
```

> 注意，字段号和字段名不能混合在一起进行保留

### 生成的文件

C++生成文件为：`.h`头文件以及`.cc`源文件

### 默认值

当解析数据时，如果没有特别的类型，不同类型的字段会有不同的默认值：

+ string，默认为空字符串
+ bytes：默认为空字节（都是使用string进行存储）
+ bool：默认为false
+ 数值类型：默认为0
+ message类型：依赖语言（C++为空类）
+ 枚举类型：默认为第一个枚举值，且必须为0

对于`repeated`字段以及`map`字段，默认为空列表或空表（依赖语言）

对于`implicit`的字段需要注意默认行为，因为像上述提到的行为，用户可能无法判断该字段是默认值，还是反序列化出来的值亦或者是根本没有提供该字段

### 枚举类型

定义一个枚举类型

```protobuf
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

#### 枚举默认值

需要注意的是，在`proto3`枚举值的第一个必须设定为0。并且推荐将第一个枚举名设置为`ENUM_TYPE_NAME_UNSPECIFIED`或者`ENUM_TYPE_NAME_UNKNOWN`。

#### 枚举值别名

同一个枚举体内，通过设置`allow_alias`为`true`，可以使多个枚举名具有相同的值。否则protoc编译器会发出警告

```protobuf
enum EnumAllowingAlias {
  option allow_alias = true;
  EAA_UNSPECIFIED = 0;
  EAA_STARTED = 1;
  EAA_RUNNING = 1;
  EAA_FINISHED = 2;
}

enum EnumNotAllowingAlias {
  ENAA_UNSPECIFIED = 0;
  ENAA_STARTED = 1;
  // ENAA_RUNNING = 1;  // 取消注释会出现警告信息
  ENAA_FINISHED = 2;
}
```

注意，枚举值是一个`int32`存储的。因为负数的效率问题，不推荐将值设置为负数。在C++中，生成的是`enum`而非`enum class`

#### 保留而非直接删除字段

同message一样，废弃字段推荐保留

```protobuf
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```

### 使用其他自定义类型

同一个IDL文件中，可以直接使用自定义的类型

```protobuf
message SearchResponse {
  repeated Result results = 1;
}
 
message Result {
  string url = 1;
  string title = 2;
  repeated string snippets = 3;
}
```

#### 导入定义

```protobuf
import "myproject/other_protos.proto";
```

### 嵌套类型

和C++中的struct一样，嵌套类的权限是public

```protobuf
message SearchResponse {
  message Result {
    string url = 1;
    string title = 2;
    repeated string snippets = 3;
  }
  repeated Result results = 1;
}

//在其他message中使用
message SomeOtherMessage {
  SearchResponse.Result result = 1;
}
```



## C++基本使用

基本流程：

+ 定义IDL文件
+ 使用protoc编译IDL文件
+ 使用API

### 定义IDL文件

```protobuf
syntax = "proto3";

package tutorial;

message Person {
  optional string name = 1;
  optional int32 id = 2;
  optional string email = 3;

  enum PhoneType {
    PHONE_TYPE_UNSPECIFIED = 0;
    PHONE_TYPE_MOBILE = 1;
    PHONE_TYPE_HOME = 2;
    PHONE_TYPE_WORK = 3;
  }

  message PhoneNumber {
    optional string number = 1;
    optional PhoneType type = 2 [default = PHONE_TYPE_HOME];
  }

  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```

第一行指明版本，默认情况下使用的是`proto2`

`package`指明命名空间，防止命名冲突

### 编译IDL文件

```shell
protoc -I=$SRC_DIR --cpp_out=$DST_DIR $SRC_DIR/addressbook.proto
```

+ `-I=$SRC_DIR`：IDL文件路径
+ `--cpp_out`：输出C++文件的地址，将会在指定位置生成`addressbook.pb.h`以及`addressbook.pb.cc`
+ `$SRC_DIR/addressbook.proto`：IDL文件

### 使用API

对于生成的C++文件，可以直接使用。每个声明的消息都会有一个对应的类，以Person类为例

```c++
  // name
  inline bool has_name() const;
  inline void clear_name();
  inline const ::std::string& name() const;
  inline void set_name(const ::std::string& value);
  inline void set_name(const char* value);
  inline ::std::string* mutable_name();

  // id
  inline bool has_id() const;
  inline void clear_id();
  inline int32_t id() const;
  inline void set_id(int32_t value);

  // email
  inline bool has_email() const;
  inline void clear_email();
  inline const ::std::string& email() const;
  inline void set_email(const ::std::string& value);
  inline void set_email(const char* value);
  inline ::std::string* mutable_email();

  // phones
  inline int phones_size() const;
  inline void clear_phones();
  inline const ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >& phones() const;
  inline ::google::protobuf::RepeatedPtrField< ::tutorial::Person_PhoneNumber >* mutable_phones();
  inline const ::tutorial::Person_PhoneNumber& phones(int index) const;
  inline ::tutorial::Person_PhoneNumber* mutable_phones(int index);
  inline ::tutorial::Person_PhoneNumber* add_phones();
```

基本上，每个成员都会有对应的setter,getter,clear以及mutable函数。mutable提供底层数据的指针。如果有一个`repeated`类型的消息成员，只会提供mutale函数，而不会提供setter

#### 枚举和内嵌类型

生成的文件中有一个`PhoneType`枚举类，使用的是旧C++的`enum`，而非17之后的`enum class`。可以通过`Person::PhoneType`来使用这个枚举类型，也可以通过`Person::PHONE_TYPE_MOBILE`，`Person::PHONE_TYPE_HOME`和`Person::PHONE_TYPE_WORK`来使用枚举值

对于内嵌类`Person::PhoneNumber`，在代码中对应的是`Person_PhoneNumber`类。唯一和C++中的内嵌类不同的是，该`Person_PhoneNumber`类可以直接进行前向声明（C++中的内嵌类则不支持）

#### 标准方法

所有的消息类都有一些统一的接口：

+ `bool IsInitialized() const`:是否初始化
+ `string DebugString() const`:调试信息
+ `void CopyFrom(const Person& from)`:通过`from`覆盖消息值
+ `void Clear()`

#### 序列化与反序列化

所有类都会根据protobuffer的二进制形式进行序列化和反序列化：

+ `bool SerializeToString(string* output) const`:将数据以二进制形式存储在string中
+ `bool ParseFromString(const string& data)`:反序列化string中的数据
+ `bool SeralizeToOstream(ostream* output) const`:将数据序列化输出到指定输出流
+ `bool ParseFromIstream(istream* input)`:从指定输入流中反序列化数据

#### 写数据

```c++
#include <iostream>
#include <fstream>
#include <string>
#include "addressbook.pb.h"
using namespace std;

// This function fills in a Person message based on user input.
void PromptForAddress(tutorial::Person* person) {
  cout << "Enter person ID number: ";
  int id;
  cin >> id;
  person->set_id(id);
  cin.ignore(256, '\n');

  //可以使用mutable方法赋值
  cout << "Enter name: ";
  getline(cin, *person->mutable_name());
  //也可以使用setter进行复制
  cout << "Enter email address (blank for none): ";
  string email;
  getline(cin, email);
  if (!email.empty()) {
    person->set_email(email);
  }
	
  while (true) {
    cout << "Enter a phone number (or leave blank to finish): ";
    string number;
    getline(cin, number);
    if (number.empty()) {
      break;
    }
	//add方法本质就是mutable+setter组合
    tutorial::Person::PhoneNumber* phone_number = person->add_phones();
    phone_number->set_number(number);

    cout << "Is this a mobile, home, or work phone? ";
    string type;
    getline(cin, type);
    if (type == "mobile") {
      phone_number->set_type(tutorial::Person::PHONE_TYPE_MOBILE);
    } else if (type == "home") {
      phone_number->set_type(tutorial::Person::PHONE_TYPE_HOME);
    } else if (type == "work") {
      phone_number->set_type(tutorial::Person::PHONE_TYPE_WORK);
    } else {
      cout << "Unknown phone type.  Using default." << endl;
    }
  }
}

// Main function:  Reads the entire address book from a file,
//   adds one person based on user input, then writes it back out to the same
//   file.
int main(int argc, char* argv[]) {
  // Verify that the version of the library that we linked against is
  // compatible with the version of the headers we compiled against.
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  if (argc != 2) {
    cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
    return -1;
  }

  tutorial::AddressBook address_book;

  {
    // Read the existing address book.
    fstream input(argv[1], ios::in | ios::binary);
    if (!input) {
      cout << argv[1] << ": File not found.  Creating a new file." << endl;
    } else if (!address_book.ParseFromIstream(&input)) {
      cerr << "Failed to parse address book." << endl;
      return -1;
    }
  }

  // Add an address.
  PromptForAddress(address_book.add_people());

  {
    // Write the new address book back to disk.
    fstream output(argv[1], ios::out | ios::trunc | ios::binary);
    if (!address_book.SerializeToOstream(&output)) {
      cerr << "Failed to write address book." << endl;
      return -1;
    }
  }

  // Optional:  Delete all global objects allocated by libprotobuf.
  google::protobuf::ShutdownProtobufLibrary();

  return 0;
}
```

注意`GOOGLE_PROTOBUF_VERIFY_VERSION`不是必须的，但是是一个好习惯，可以对本地库进行检查，防止使用了不兼容的库

`google::protobuf::ShutdownProtobufLibrary();`也不是必须的，因为当程序结束，操作系统会回收所有内存，但是也是一个好习惯。

#### 读取信息

```c++
#include <iostream>
#include <fstream>
#include <string>
#include "addressbook.pb.h"
using namespace std;

// Iterates though all people in the AddressBook and prints info about them.
void ListPeople(const tutorial::AddressBook& address_book) {
  for (int i = 0; i < address_book.people_size(); i++) {
    const tutorial::Person& person = address_book.people(i);

    cout << "Person ID: " << person.id() << endl;
    cout << "  Name: " << person.name() << endl;
    if (person.has_email()) {
      cout << "  E-mail address: " << person.email() << endl;
    }

    for (int j = 0; j < person.phones_size(); j++) {
      const tutorial::Person::PhoneNumber& phone_number = person.phones(j);

      switch (phone_number.type()) {
        case tutorial::Person::PHONE_TYPE_MOBILE:
          cout << "  Mobile phone #: ";
          break;
        case tutorial::Person::PHONE_TYPE_HOME:
          cout << "  Home phone #: ";
          break;
        case tutorial::Person::PHONE_TYPE_WORK:
          cout << "  Work phone #: ";
          break;
      }
      cout << phone_number.number() << endl;
    }
  }
}

// Main function:  Reads the entire address book from a file and prints all
//   the information inside.
int main(int argc, char* argv[]) {
  // Verify that the version of the library that we linked against is
  // compatible with the version of the headers we compiled against.
  GOOGLE_PROTOBUF_VERIFY_VERSION;

  if (argc != 2) {
    cerr << "Usage:  " << argv[0] << " ADDRESS_BOOK_FILE" << endl;
    return -1;
  }

  tutorial::AddressBook address_book;

  {
    // Read the existing address book.
    fstream input(argv[1], ios::in | ios::binary);
    if (!address_book.ParseFromIstream(&input)) {
      cerr << "Failed to parse address book." << endl;
      return -1;
    }
  }

  ListPeople(address_book);

  // Optional:  Delete all global objects allocated by libprotobuf.
  google::protobuf::ShutdownProtobufLibrary();

  return 0;
}
```

#### 扩展

如果后续想要更改字段，为了保持兼容性，必须遵守以下原则：

+ 不能改变现存字段的号码
+ 不能添加或删除任何`required`字段
+ 可以删除或者添加`optional`或者`repeated`字段，如果添加，必须使用全新的字段号码（包括被删除的字段号码也不能使用）

遵守这些原则，旧代码会忽略新的字段，而新代码可以很好的兼容旧代码

#### 性能压榨

尽管protobuf已经经过了性能优化，但是还可以通过以下手段进行压榨：

+ 尽可能的重用消息对象。因为消息一旦被创建，即使clear过了依旧会保留内存。因此在处理大量重复数据的时候，重用消息对象会带来不小的性能提升。在使用消息对象时，应该使用`SpaceUsed`方法监控消息对象的大小
+ 当在多线程分配大量小对象时，系统的malloc函数可能不够优秀，可以使用[Google’s TCMalloc](https://github.com/google/tcmalloc)