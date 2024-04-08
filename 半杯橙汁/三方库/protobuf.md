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