# fast DDS

> 仅C++版本

## ch01 开始

### 1.1 什么是DDS

数据分发服务(Data Distribution Service,DDS)是为分布式软件通信提供的一种以数据为中心的通信协议。该协议描述了支持数据提供者(data providers)和数据消费者(data consumers)之间通信的通信接口（APIs）以及通信语义(Communication Semantics)

因为DDS是一个以数据为中心的发布订阅模型(Data-Centric Publish Subscribe model,aka DCPS),所以它的实现中有三个应用实体：

+ 发布实体：负责定义信息生成对象以及属性
+ 订阅实体：负责定义信息消费对象以及属性
+ 配置实体：定义作为话题传输的消息类型，以及根据消息质量(Quality of Service,aka QoS)创建发布者和订阅者，并确保以上实体有正确的性能（即创建发布者和订阅者，并提供可靠的QoS）

DDS使用QoS来定义DDS实体的行为特征。QoS由独立的QoS策略（由QoSPolicy派生出来的类型对象）组成。这些内容在3.1.2策略章节描述

#### 1.1.1 DCPS概念模型

在DCPS模型中，为通信应用程序系统定义了四个基础元素：

+ **发布者(**Publisher)：DCPS中负责创建和配置*写数据者*(DataWriters)的实体。写数据者(DataWriters)是实际上负责发送数据的实体。每个实体都将在指定的话题(Topic)下发送数据。更多细节在3.3章节叙述
+ **订阅者**(Subscriber):DCPS中负责接收自己所订阅话题下的信息的实体。它提供一个或多个*数据接受者*(DataReader)对象,负责将新数据的有效性传递给应用程序。更多细节在3.4章节叙述
+ **话题**(Topic):DCPS中绑定发布和订阅的实体。在一个DDS范围内，话题是唯一的。通过*话题描述*(TopicDescription)，它统一了发布和订阅的数据类型。更多细节在3.5章节叙述
+ 域(Domain):这是用来连接所有在相同或者不同应用程序中，通过不同话题进行数据交换的发布者和订阅者的概念。参加同一个*域*的独立程序被称作*域成员*(DomainParticipant)。DDS域由域ID进行识别。域成员通过定义域ID来指定自己所属域。在网络中，两个不同ID的域成员，彼此并不知晓对方的存在。因此，可能会创建多条通信通道。这些是为了以下场景所设计的：当存在多个DDS应用程序参与时，彼此的域成员在相互交互时，不同的DDS应用程序不得相互干扰。域成员对于其他DCPS实体来说像一个容器，对于发布者、订阅者以及话题实体来说像一个工厂，并在域中提供管理服务。更多细节在3.2章节叙述

DDS域中的DCPS模型实体示例图如下：

![image-20241011164039831](./images/image-20241011164039831.png)

### 1.2 什么是RTPS

*实时发布订阅*(Real-Time Publish Subscribe)协议是依赖最大努力交付传输协议（如UDP/IP），为DDS应用设计的一个发布-订阅通信中间件。此外，Fast DDS还提供TCP以及共享内存(Shared Memory,aka SHM)传输方式。

该协议是为了支持单播以及多播通信设计的。

RTPS的最顶层定义了一个单独的通信层，这点继承自DDS的域(Domain)概念。多个域可以在同一时间独立的共存。一个域包含任意数量的*RTPS成员*（RTPSParticipants,能够发送和接收数据）。为了实现这些，RTPS成员使用它们的*端点*(Endpoints):

+ RTPSWriter:发送数据的端点
+ RTPSReader:接收数据的端点

一个RTPS成员可以由任意数量的读写端点，RTPS抽象图如下：

![image-20241011165433962](./images/image-20241011165433962.png)

通信围绕*话题*(Topics)展开，话题定义和标记要交换的数据。话题并不属于某个特定的成员。成员通过*写端点*(RTPSWriter)对某个话题下发布的数据进行修改，并通过*读端点*(RTPSReader)在相应的话题下接收该数据。通信单元称之为*变化*(Change),代表某个话题下的数据中的一次更新。读写端点在自己的*历史*(History，作为存储最近变化的一个缓存服务)中注册这些变化

在默认配置的Fast DDS中，当你使用写端点（RTPSWriter）发布一个变化，将发生以下步骤：

+ 将变化添加至写端点的历史中
+ 写端点将变化发送给所有自己记录的读端点
+ 在读端点接收到数据后，读端点根据新变化更新自己的历史缓存

但是，Fast DDS支持多种多样的配置，允许你更改读写端点的行为。对默认的RTPS实体配置进行修改，意味着读写端点之间的数据交换流将会发生改变。此外，你可以通过选择服务质量(QoS)策略来影响如何管理历史缓存，但通信流程不会变化。FastDDS中如何实现RTPS协议的细节，可以在第四章继续阅读。

### 1.3 写一个简单的C++发布者订阅者应用

本节手把手教学如何使用C++ API创建一个FastDDS的发布者订阅者应用。也可以使用Fast DDS-Gen工具自动生成该小节实现的应用。使用Fast DDS-Gen的方法在第三章介绍。

#### 1.3.1 背景

DDS是实现了DCPS模型的以数据为中心的通信中间件。该模型基于发布者（数据生产者）和订阅者（数据消费者）。这些实体通过话题（话题绑定上述两个实体，即绑定发布者和订阅者）进行通信。发布者在话题下发布数据，订阅者在同一话题下接收数据

#### 1.3.2 前置条件

首先，你需要根据安装手册安装eProsima Fast DDS以及其所有依赖。同时也要安装好eProsima Fast DDS-Gen工具。注意，本教程的所有命令都是基于Linux系统的。

#### 1.3.3 创建工作空间

应用工作空间的最后的结构应该如下。`build/DDSHelloWorldPublisher`和`build/DDSHelloWorldSubscriber`分别是发布者和订阅者的应用程序：

![image-20241012103045781](./images/image-20241012103045781.png)

首先，创建目录树：

```shell
mkdir workspace_DDSHelloWorld && cd workspace_DDSHelloWorld
mkdir src build
```

#### 1.3.4 导入依赖以及链接库

DDS应用依赖Fast DDS以及Fast CDR库。不同环境安装这些依赖的步骤会有所不同。

##### 1.3.4.1 通过可执行文件安装和手动安装

如果已经通过可执行文件安装或者手动安装，这些库应该已经可以从工作空间进行访问。在Linux下，Fast DDS以及Fast CDR头文件应该分别在`/usr/include/fastdds`以及`/usr/include/fastcdr`。链接库应该在`/usr/lib`。（橙子注：官网教程中，源码安装的过程并没有直接安装在系统目录下，而是安装在了`~/Fast-DDS`目录下，所以对应头文件的目录应该是`~/Fast-DDS/install/include/fastdds`和`~/Fast-DDS/install/include/fastcdr`，链接库文件应该在`~/FastDDS/install/lib`，为了方便，最好重新编译并安装至系统目录）

##### 1.3.4.2 Colcon安装

> 橙子注：同ROS1的catkin一样，是CMake的更高一层封装

如果安装了Colcon，可以通过不同方式导入库。如果仅需要再当前会话中导入库，可执行一下命令：

```c++
source <path/to/Fast-DDS/workspace>/install/setup.bash
```

可以通过在使用以下命令，在shell配置文件中，将Fast DDS安装目录添加系统环境变量`$PATH`，使Fast DDS对所有会话生效：

```shell
echo 'source <path/to/Fast-DDS/workspace>/install/setup.bash' >> ~/.bashrc
```

系统变量将在每次用户登录后生效。

#### 1.3.5 配置CMake项目

教程中将使用原生的CMake管理和构建项目文件。使用自己顺手的文本编辑器，创建一个新的`CMakeLists.txt`并将下列代码粘贴进去。在自己工作空间的根目录下保存该文件。如果完全安装教程走的，根目录应该为`~/workspace_DDSHelloWorld`

```CMake
cmake_minimum_required(VERSION 3.20)

project(DDSHelloWorld)

# 寻找依赖库，注意，这里是安装在了系统目录下。
if(NOT fastcdr_FOUND)
    find_package(fastcdr 2 REQUIRED)
endif()

if(NOT fastdds_FOUND)
    find_package(fastdds 3 REQUIRED)
endif()

# 编译选项设置语言标准为C++11
include(CheckCXXCompilerFlag)
if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_COMPILER_IS_CLANG OR
        CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    check_cxx_compiler_flag(-std=c++11 SUPPORTS_CXX11)
    if(SUPPORTS_CXX11)
        add_compile_options(-std=c++11)
    else()
        message(FATAL_ERROR "Compiler doesn't support C++11")
    endif()
endif()

message(STATUS "Configuring HelloWorld publisher/subscriber example...")
# src下所有.cxx文件作为源文件
file(GLOB DDS_HELLOWORLD_SOURCES_CXX "src/*.cxx")
```

后续会根据需要的头文件或者生成文件，逐步完善该文件

#### 1.3.6 构造话题数据类型

eProsima Fast DDS-Gen是一个用来通过*接口定义语言*(Interface Description Language,aka ID)生成源码的Java程序。这个程序可以做两件事：

+ 为自定义话题信息生成C++定义
+ 使用你的话题数据生成一个功能示例

教程将使用第一个功能。后者的应用示例可以在第三章看到。在Fast DDS-Gen的介绍中会有更详细的说明。本项目中，将使用Fast DDS-Gen定义用来发送和接收的话题数据类型。

在工作空间中，执行以下命令：

```shell
cd src && touch HelloWorld.idl
```

执行后将会在`src`目录下创建`HelloWorld.idl`文件。在文本编辑器下打开，并将下属内容粘贴进去：

```c++
struct HelloWorld
{
    unsigned long index;
    string message;
};
```

这个文件定义了`HelloWorld`数据类型，该类型有两个数据成员：一个`uint32_t`类型的index，以及一个`std::string`类型的message。剩下的工作就是在C++11中生成源代码。可以在`src`目录下执行（）：

```shell
<path/to/Fast DDS-Gen>/scripts/fastddsgen HelloWorld.idl
# 如果已经设置了环境变量
fastddsgen HelloWorld.idl
```

执行后会生成以下文件（橙子注：类似protobuf中的生成器）：

+ HelloWorld.hpp:HelloWorld类型定义
+ HelloWorldPubSubTypes.cxx:Fast DDS使用HelloWorld类型的接口
+ HelloWorldPubSubTypes.h:HelloWorldPubSubTypes.cxx的头文件
+ HelloWorldCdrAux.ipp:HelloWorld类型序列化以及反序列化代码
+ HelloWorldCdrAux.hpp:HelloWorldCdrAux.ipp的头文件
+ HelloWorldTypeObjectSupport.cxx:类型对象注册代码
+ HelloWorldTypeObjectSupport.hpp:HelloWorldTypeObjectSupport.cxx的头文件

##### 1.3.6.1 CMakeLists

在刚刚创建的`CMakeLists.txt`最后添加以下语句：

> 橙子注：这里如果在IDE或者vscode(配置了CMake插件的情况下)中添加会飘红，因为此时还不存在目标DDSHelloWorldPublisher。这里可以跳过，无伤大雅

```cmake
target_link_libraries(DDSHelloWorldPublisher fastdds fastcdr)
```

#### 1.3.7 编写Fast DDS发布者

在工作空间的`src`目录下，执行以下代码获取源文件：

```shell
wget -O HelloWorldPublisher.cpp \
    https://raw.githubusercontent.com/eProsima/Fast-RTPS-docs/master/code/Examples/C++/DDSHelloWorld/src/HelloWorldPublisher.cpp
```

下载后代码如下：

```c++
// Copyright 2016 Proyectos y Sistemas de Mantenimiento SL (eProsima).
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/**
 * @file HelloWorldPublisher.cpp
 *
 */

#include "HelloWorldPubSubTypes.hpp"

//standard cpp
#include <chrono>
#include <thread>


//third party 
#include <fastdds/dds/domain/DomainParticipant.hpp>
#include <fastdds/dds/domain/DomainParticipantFactory.hpp>
#include <fastdds/dds/publisher/DataWriter.hpp>
#include <fastdds/dds/publisher/DataWriterListener.hpp>
#include <fastdds/dds/publisher/Publisher.hpp>
#include <fastdds/dds/topic/TypeSupport.hpp>

//alias
using namespace eprosima::fastdds::dds;

class HelloWorldPublisher
{
private:
    ///@brief 消息的实际载体
    HelloWorld hello_;
    ///@brief 域成员
    DomainParticipant* participant_;
    /// @brief 发布者
    Publisher* publisher_;
    /// @brief 话题
    Topic* topic_;
    /// @brief 数据发送者
    DataWriter* writer_;
    /// @brief 类型管理者（负责消息类型的注册）
    TypeSupport type_;
    /// @brief 数据发送者的监听者，主要负责设置对应事件的回调，本例子中设置了发布消息是否配对，即存在订阅者订阅了该话题的该信息(matched)
    class PubListener : public DataWriterListener
    {
    public:

        PubListener()
            : matched_(0)
        {
        }

        ~PubListener() override
        {
        }

        void on_publication_matched(
                DataWriter*,
                const PublicationMatchedStatus& info) override
        {
            if (info.current_count_change == 1)
            {
                matched_ = info.total_count;
                std::cout << "Publisher matched." << std::endl;
            }
            else if (info.current_count_change == -1)
            {
                matched_ = info.total_count;
                std::cout << "Publisher unmatched." << std::endl;
            }
            else
            {
                std::cout << info.current_count_change
                        << " is not a valid value for PublicationMatchedStatus current count change." << std::endl;
            }
        }

        std::atomic_int matched_;

    } listener_;

public:

    HelloWorldPublisher()
        : participant_(nullptr)
        , publisher_(nullptr)
        , topic_(nullptr)
        , writer_(nullptr)
        , type_(new HelloWorldPubSubType())
    {
    }
    
    virtual ~HelloWorldPublisher()
    {
        if (writer_ != nullptr)
        {
            publisher_->delete_datawriter(writer_);
        }
        if (publisher_ != nullptr)
        {
            participant_->delete_publisher(publisher_);
        }
        if (topic_ != nullptr)
        {
            participant_->delete_topic(topic_);
        }
        DomainParticipantFactory::get_instance()->delete_participant(participant_);
    }

    //!Initialize the publisher
    bool init()
    {
        //设置消息内容
        hello_.index(0);
        hello_.message("HelloWorld");
        //设置域成员的服务质量，并创建域成员实体
        DomainParticipantQos participantQos;
        participantQos.name("Participant_publisher");
        participant_ = DomainParticipantFactory::get_instance()->create_participant(0, participantQos);

        if (participant_ == nullptr)
        {
            return false;
        }

        // Register the Type
        type_.register_type(participant_);

        // Create the publications Topic
        topic_ = participant_->create_topic("HelloWorldTopic", "HelloWorld", TOPIC_QOS_DEFAULT);

        if (topic_ == nullptr)
        {
            return false;
        }

        // Create the Publisher
        publisher_ = participant_->create_publisher(PUBLISHER_QOS_DEFAULT, nullptr);

        if (publisher_ == nullptr)
        {
            return false;
        }

        // Create the DataWriter
        writer_ = publisher_->create_datawriter(topic_, DATAWRITER_QOS_DEFAULT, &listener_);

        if (writer_ == nullptr)
        {
            return false;
        }
        return true;
    }

    //!Send a publication
    bool publish()
    {
        if (listener_.matched_ > 0)
        {
            hello_.index(hello_.index() + 1);
            writer_->write(&hello_);
            return true;
        }
        return false;
    }

    //!Run the Publisher
    void run(
            uint32_t samples)
    {
        uint32_t samples_sent = 0;
        while (samples_sent < samples)
        {
            if (publish())
            {
                samples_sent++;
                std::cout << "Message: " << hello_.message() << " with index: " << hello_.index()
                            << " SENT" << std::endl;
            }
            std::this_thread::sleep_for(std::chrono::milliseconds(1000));
        }
    }
};

int main(
        int argc,
        char** argv)
{
    std::cout << "Starting publisher." << std::endl;
    uint32_t samples = 10;

    HelloWorldPublisher* mypub = new HelloWorldPublisher();
    if(mypub->init())
    {
        mypub->run(samples);
    }

    delete mypub;
    return 0;
}
```

橙子注：贴图帮助理解，该实例中多了DataWriterListener,该类是数据生产者的监听者，可以用来对数据生产者的对应事件设置回调，本例子中是对话题发布成功设置了回调函数。

![image-20241012135250379](./images/image-20241012135250379.png)

##### 1.3.7.1 代码解释

开头是Doxygen风格的注释内容，关键字file输命令文件名称

```C++
/**
 * @file HelloWorldPublisher.cpp
 *
 */
```

接着是C++的头文件。第一个导入的`HelloWorldPubSubTypes.h`包含了自定义类型的序列化和反序列化功能。

```c++
#include "HelloWorldPubSubTypes.hpp"
```

接下来是C++标准库`chrono`和`thread`以及fastDDS的API文件

+ DomainParticipantFactory:创建和销毁域成员
+ DomainParticipant:作为其他所有实体的容器存在，也作为发布者、订阅者以及话题的工厂
+ TypeSupport:向成员提供序列化、反序列化以及获取指定数据类型的关键字的功能
+ Publisher:负责创建DataWriters
+ DataWriters:允许用户对给定话题下的数据进行复制
+ DataWriterListener:允许对DataWriterListener的函数重定义

```c++
#include <chrono>
#include <thread>

#include <fastdds/dds/domain/DomainParticipant.hpp>
#include <fastdds/dds/domain/DomainParticipantFactory.hpp>
#include <fastdds/dds/publisher/DataWriter.hpp>
#include <fastdds/dds/publisher/DataWriterListener.hpp>
#include <fastdds/dds/publisher/Publisher.hpp>
#include <fastdds/dds/topic/TypeSupport.hpp>
```

接下来是使用fastdds的命名空间：

```c++
using namespace eprosima::fastdds::dds;
```

接下来是`HelloWorldPublisher`类的实现

```c++
class HelloWorldPublisher
```

接下来是私有数据成员定义：

```c++
private:
    ///@brief 消息的实际载体
    HelloWorld hello_;
    ///@brief 域成员
    DomainParticipant* participant_;
    /// @brief 发布者
    Publisher* publisher_;
    /// @brief 话题
    Topic* topic_;
    /// @brief 数据发送者
    DataWriter* writer_;
    /// @brief 类型管理者（负责消息类型的注册）
    TypeSupport type_;
```

接下来，`PubListener`继承自`DataWriterListener`。该类重写了一些DataWriter指定事件的回调函数。重载的函数`on_publication_matched()`允许定义当DataWriter发出的话题被某个DataReader监听时（即match）后的一系列行为。`info.current_count_change()`检测配对的DataReader**s**和DataWriter之间变化。这是`MatchedStatus`结构中的一个成员，可以用来追踪一个订阅的变化。最后`listener_`对象是`PubListener`的一个实例

```c++
class PubListener : public DataWriterListener
{
public:

    PubListener()
        : matched_(0)
    {
    }

    ~PubListener() override
    {
    }

    void on_publication_matched(
            DataWriter*,
            const PublicationMatchedStatus& info) override
    {
        if (info.current_count_change == 1)
        {
            matched_ = info.total_count;
            std::cout << "Publisher matched." << std::endl;
        }
        else if (info.current_count_change == -1)
        {
            matched_ = info.total_count;
            std::cout << "Publisher unmatched." << std::endl;
        }
        else
        {
            std::cout << info.current_count_change
                    << " is not a valid value for PublicationMatchedStatus current count change." << std::endl;
        }
    }

    std::atomic_int matched_;

} listener_;
```

`HelloWorldPublisher`的构造和析构定义如下。构造者将除了TypeSupprot对象外的所有的私有成员初始化为空指针`nullptr`，TypeSupport对象则初始化为`HelloWorldPubSunType`的一个实例。析构者负责回收内存。

```c++
HelloWorldPublisher()
    : participant_(nullptr)
    , publisher_(nullptr)
    , topic_(nullptr)
    , writer_(nullptr)
    , type_(new HelloWorldPubSubType())
{
}

virtual ~HelloWorldPublisher()
{
    if (writer_ != nullptr)
    {
        publisher_->delete_datawriter(writer_);
    }
    if (publisher_ != nullptr)
    {
        participant_->delete_publisher(publisher_);
    }
    if (topic_ != nullptr)
    {
        participant_->delete_topic(topic_);
    }
    DomainParticipantFactory::get_instance()->delete_participant(participant_);
}
```

接下来是`HelloWorldPublisher`类的公共成员方法，定义了发布者的初始化函数。该函数有以下作用：

+ 初始化HelloWorld类型成员`hello_`的的数据成员
+ 通过域成员的QoS为成员分配一个名称
+ 通过`DomainParticipantFactory`创建一个域成员
+ 在IDL中注册数据类型
+ 为发布者创建一个话题
+ 创建发布者
+ 创建DataWriter以及之前定义的监听者

如你所见，除了域成员的名字，QoS对所有实体设置了默认配置(`PARTICIPANT_QOS_DEFAULT`,`PUBLISHER_QOS_DEFAULT`,`TOPIC_QOS_DEFAULT`,`DATAWRITER_QOS_DEFAULT`)。每个实体的QoS值可以在[DDS标准](https://www.omg.org/spec/DDS/About-DDS/)中查看

```c++
//!Initialize the publisher
bool init()
{
    hello_.index(0);
    hello_.message("HelloWorld");

    DomainParticipantQos participantQos;
    participantQos.name("Participant_publisher");
    participant_ = DomainParticipantFactory::get_instance()->create_participant(0, participantQos);

    if (participant_ == nullptr)
    {
        return false;
    }

    // Register the Type
    type_.register_type(participant_);

    // Create the publications Topic
    topic_ = participant_->create_topic("HelloWorldTopic", "HelloWorld", TOPIC_QOS_DEFAULT);

    if (topic_ == nullptr)
    {
        return false;
    }

    // Create the Publisher
    publisher_ = participant_->create_publisher(PUBLISHER_QOS_DEFAULT, nullptr);

    if (publisher_ == nullptr)
    {
        return false;
    }

    // Create the DataWriter
    writer_ = publisher_->create_datawriter(topic_, DATAWRITER_QOS_DEFAULT, &listener_);

    if (writer_ == nullptr)
    {
        return false;
    }
    return true;
}
```

我们实现了成员函数`publish()`用以发布话题。在DataWriter的监听者中，会根据发布话题下DataWriter和DataReader是否有匹配来更新`matched_`的值。该值包含了有多少个DataReaders被发现。因此，只要有一个DataReader被发现，应用程序就开始发布话题。这就是DataWriter对象中变化(change,DDS术语)的写动作。

```c++
bool publish()
{
    if (listener_.matched_ > 0)
    {
        hello_.index(hello_.index() + 1);
        writer_->write(&hello_);
        return true;
    }
    return false;
}
```

`run`方法根据参数发布指定次数的话题。

```c++
//!Run the Publisher
void run(
        uint32_t samples)
{
    uint32_t samples_sent = 0;
    while (samples_sent < samples)
    {
        if (publish())
        {
            samples_sent++;
            std::cout << "Message: " << hello_.message() << " with index: " << hello_.index()
                        << " SENT" << std::endl;
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(1000));
    }
}
```

最后，在主函数中运行`HelloWorldPublisher`

```c++
int main(
        int argc,
        char** argv)
{
    std::cout << "Starting publisher." << std::endl;
    uint32_t samples = 10;

    HelloWorldPublisher* mypub = new HelloWorldPublisher();
    if(mypub->init())
    {
        mypub->run(samples);
    }

    delete mypub;
    return 0;
}
```

##### 1.3.7.2 CMakeLists.txt

在之前创建的CMakeList.txt中粘贴以下内容。这些命令将需要构造可执行文件的依赖库以及源文件均整合在一起。

```cmake
add_executable(DDSHelloWorldPublisher src/HelloWorldPublisher.cpp ${DDS_HELLOWORLD_SOURCES_CXX})
target_link_libraries(DDSHelloWorldPublisher fastdds fastcdr)
```

此时，该项目已经支持构建，编译和运行发布者程序了。在build目录下执行以下语句：

```shell
cmake .. && cmake --build .
./DDSHelloWorldPublisher
```

#### 1.3.8 编写订阅者

在工作空间的`src`目录下，执行以下代码获取源文件：

```shell
wget -O HelloWorldSubscriber.cpp https://raw.githubusercontent.com/eProsima/Fast-RTPS-docs/master/code/Examples/C++/DDSHelloWorld/src/HelloWorldSubscriber.cpp
```

下载后代码如下：

```c++
// Copyright 2016 Proyectos y Sistemas de Mantenimiento SL (eProsima).
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

/**
 * @file HelloWorldSubscriber.cpp
 *
 */

#include "HelloWorldPubSubTypes.hpp"

#include <chrono>
#include <thread>

#include <fastdds/dds/domain/DomainParticipant.hpp>
#include <fastdds/dds/domain/DomainParticipantFactory.hpp>
#include <fastdds/dds/subscriber/DataReader.hpp>
#include <fastdds/dds/subscriber/DataReaderListener.hpp>
#include <fastdds/dds/subscriber/qos/DataReaderQos.hpp>
#include <fastdds/dds/subscriber/SampleInfo.hpp>
#include <fastdds/dds/subscriber/Subscriber.hpp>
#include <fastdds/dds/topic/TypeSupport.hpp>

using namespace eprosima::fastdds::dds;

class HelloWorldSubscriber
{
private:
    /// @brief 域成员
    DomainParticipant* participant_;
    /// @brief 订阅者
    Subscriber* subscriber_;
    /// @brief 数据读取端点endpoint
    DataReader* reader_;
    /// @brief 话题
    Topic* topic_;
    /// @brief 类型管理者
    TypeSupport type_;
    /// @brief  数据接收监听者
    class SubListener : public DataReaderListener
    {
    public:

        SubListener()
            : samples_(0)
        {
        }

        ~SubListener() override
        {
        }

        void on_subscription_matched(
                DataReader*,
                const SubscriptionMatchedStatus& info) override
        {
            if (info.current_count_change == 1)
            {
                std::cout << "Subscriber matched." << std::endl;
            }
            else if (info.current_count_change == -1)
            {
                std::cout << "Subscriber unmatched." << std::endl;
            }
            else
            {
                std::cout << info.current_count_change
                          << " is not a valid value for SubscriptionMatchedStatus current count change" << std::endl;
            }
        }

        void on_data_available(
                DataReader* reader) override
        {
            SampleInfo info;
            if (reader->take_next_sample(&hello_, &info) == RETCODE_OK)
            {
                if (info.valid_data)
                {
                    samples_++;
                    std::cout << "Message: " << hello_.message() << " with index: " << hello_.index()
                              << " RECEIVED." << std::endl;
                }
            }
        }

        HelloWorld hello_;

        std::atomic_int samples_;

    }
    listener_;

public:

    HelloWorldSubscriber()
        : participant_(nullptr)
        , subscriber_(nullptr)
        , topic_(nullptr)
        , reader_(nullptr)
        , type_(new HelloWorldPubSubType())
    {
    }

    virtual ~HelloWorldSubscriber()
    {
        if (reader_ != nullptr)
        {
            subscriber_->delete_datareader(reader_);
        }
        if (topic_ != nullptr)
        {
            participant_->delete_topic(topic_);
        }
        if (subscriber_ != nullptr)
        {
            participant_->delete_subscriber(subscriber_);
        }
        DomainParticipantFactory::get_instance()->delete_participant(participant_);
    }

    //!Initialize the subscriber
    bool init()
    {
        DomainParticipantQos participantQos;
        participantQos.name("Participant_subscriber");
        participant_ = DomainParticipantFactory::get_instance()->create_participant(0, participantQos);

        if (participant_ == nullptr)
        {
            return false;
        }

        // Register the Type
        type_.register_type(participant_);

        // Create the subscriptions Topic
        topic_ = participant_->create_topic("HelloWorldTopic", "HelloWorld", TOPIC_QOS_DEFAULT);

        if (topic_ == nullptr)
        {
            return false;
        }

        // Create the Subscriber
        subscriber_ = participant_->create_subscriber(SUBSCRIBER_QOS_DEFAULT, nullptr);

        if (subscriber_ == nullptr)
        {
            return false;
        }

        // Create the DataReader
        reader_ = subscriber_->create_datareader(topic_, DATAREADER_QOS_DEFAULT, &listener_);

        if (reader_ == nullptr)
        {
            return false;
        }

        return true;
    }

    //!Run the Subscriber
    void run(
            uint32_t samples)
    {
        while (listener_.samples_ < samples)
        {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }

};

int main(
        int argc,
        char** argv)
{
    std::cout << "Starting subscriber." << std::endl;
    uint32_t samples = 10;

    HelloWorldSubscriber* mysub = new HelloWorldSubscriber();
    if (mysub->init())
    {
        mysub->run(samples);
    }

    delete mysub;
    return 0;
}
```

##### 1.3.8.1 代码解释

由于发布者和订阅者应用存在大量重复，所以本节重点关注不同点，忽略已经解释过的代码

和发布者一样，先添加FastDDS依赖的头文件，在订阅者，这些头文件被替换成了这些类：

+ Subscriber:负责创建和配置DataReaders
+ DataReader:实际负责接收数据的对象。它在应用程序中通过话题注册订阅者需要接收和访问的数据
+ DataReaderListener:分配给DataReader的监听者
+ DataReaderQoS:DataReader的QoS
+ SampleInfo:每个被“读取”或者“采集”的数据附带的信息。

```c++
#include <fastdds/dds/domain/DomainParticipant.hpp>
#include <fastdds/dds/domain/DomainParticipantFactory.hpp>
#include <fastdds/dds/subscriber/DataReader.hpp>
#include <fastdds/dds/subscriber/DataReaderListener.hpp>
#include <fastdds/dds/subscriber/qos/DataReaderQos.hpp>
#include <fastdds/dds/subscriber/SampleInfo.hpp>
#include <fastdds/dds/subscriber/Subscriber.hpp>
#include <fastdds/dds/topic/TypeSupport.hpp>
```

接下来是`HelloWorldSubscriber`的实现

```c++
class HelloWorldSubscriber
```

在私有数据成员中，DataReaderListener值得注意。私有数据成员有：域成员，话题，DataReader，以及DataType。和发布者中的DataWriter一样，监听者一样实现了对对应事件的回调函数。SubListener第一个重载的函数是`on_subscription_matched()`,对应的是DataWriter中`on_publication_matched()`回调

```c++
void on_subscription_matched(
        DataReader*,
        const SubscriptionMatchedStatus& info) override
{
    if (info.current_count_change == 1)
    {
        std::cout << "Subscriber matched." << std::endl;
    }
    else if (info.current_count_change == -1)
    {
        std::cout << "Subscriber unmatched." << std::endl;
    }
    else
    {
        std::cout << info.current_count_change
                  << " is not a valid value for SubscriptionMatchedStatus current count change" << std::endl;
    }
}
```

第二个重载的回调是`on_data_available()`。在这里，下一个可以被DataReader读取的数据将被在这里接收以及处理，并展示数据内容。`SampleInfo`类在这里定义，该类决定一个样本(sample)是否已经被读取或者接收。每读到一个样本，读取样本的计数就会增加(samples_++)。

```c++
void on_data_available(
        DataReader* reader) override
{
    SampleInfo info;
    if (reader->take_next_sample(&hello_, &info) == RETCODE_OK)
    {
        if (info.valid_data)
        {
            samples_++;
            std::cout << "Message: " << hello_.message() << " with index: " << hello_.index()
                      << " RECEIVED." << std::endl;
        }
    }
}
```

发布者的构造和析构如下：

```c++
    HelloWorldSubscriber()
        : participant_(nullptr)
        , subscriber_(nullptr)
        , topic_(nullptr)
        , reader_(nullptr)
        , type_(new HelloWorldPubSubType())
    {
    }

    virtual ~HelloWorldSubscriber()
    {
        if (reader_ != nullptr)
        {
            subscriber_->delete_datareader(reader_);
        }
        if (topic_ != nullptr)
        {
            participant_->delete_topic(topic_);
        }
        if (subscriber_ != nullptr)
        {
            participant_->delete_subscriber(subscriber_);
        }
        DomainParticipantFactory::get_instance()->delete_participant(participant_);
    }
```

接下来是订阅者的初始化公共方法。和`HelloWorldPublisher`的初始化方法一样。除了域成员的名字，其他实体的QoS都是默认配置(`PARTICIPIANT_QOS_DEFAULT`,`SUBSCRIBER_QOS_DEFAULT`,`TOPIC_QOS_DEFAULT`,`DATAREADER_QOS_DEFAULT`),每个实体的QoS值可以在[DDS标准](https://www.omg.org/spec/DDS/About-DDS/)中查看。

```c++
//!Initialize the subscriber
bool init()
{
    DomainParticipantQos participantQos;
    participantQos.name("Participant_subscriber");
    participant_ = DomainParticipantFactory::get_instance()->create_participant(0, participantQos);

    if (participant_ == nullptr)
    {
        return false;
    }

    // Register the Type
    type_.register_type(participant_);

    // Create the subscriptions Topic
    topic_ = participant_->create_topic("HelloWorldTopic", "HelloWorld", TOPIC_QOS_DEFAULT);

    if (topic_ == nullptr)
    {
        return false;
    }

    // Create the Subscriber
    subscriber_ = participant_->create_subscriber(SUBSCRIBER_QOS_DEFAULT, nullptr);

    if (subscriber_ == nullptr)
    {
        return false;
    }

    // Create the DataReader
    reader_ = subscriber_->create_datareader(topic_, DATAREADER_QOS_DEFAULT, &listener_);

    if (reader_ == nullptr)
    {
        return false;
    }

    return true;
```

`run()`方法确保所有的样本被接收。这个方法使用忙等待，为了缓解CPU使用100ms的睡眠间隔

```c++
    void run(
            uint32_t samples)
    {
        while (listener_.samples_ < samples)
        {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }
```

最后在main中执行订阅者。

```c++
int main(
        int argc,
        char** argv)
{
    std::cout << "Starting subscriber." << std::endl;
    uint32_t samples = 10;

    HelloWorldSubscriber* mysub = new HelloWorldSubscriber();
    if (mysub->init())
    {
        mysub->run(samples);
    }

    delete mysub;
    return 0;
}
```

##### 1.3.8.2 CMakeLists.txt

依旧是在CMakeLists.txt中添加以下代码：

```cmake
add_executable(DDSHelloWorldSubscriber src/HelloWorldSubscriber.cpp ${DDS_HELLOWORLD_SOURCES_CXX})
target_link_libraries(DDSHelloWorldSubscriber fastdds fastcdr)
```

此时，项目已经可以构建编译和运行订阅者程序了。在build目录下执行以下语句

```shell
cmake .. && cmake --build .
./DDSHelloWorldSubscriber
```

#### 1.3.9 执行程序

开启两个终端，分别执行发布者和订阅者：

![image-20241012154151833](./images/image-20241012154151833.png)

#### 1.3.10 总结

本节讲解了基础的DDS发布者和订阅者项目的搭建以及CMake文件编写以及添加依赖库等。

#### 1.3.11 下一步

在eProsima Fast DDS Github中可以找到更多针对不同用例和场景的复杂示例。可以在[这里](https://github.com/eProsima/Fast-DDS/tree/master/examples/cpp)找到

## ch02 库概览

Fast DDS（前称Fast RTPS）是一个高效高性能的DDS规范的实现，一个为分布式软件设计的DCPS中间件。本节内容回顾Fast DDS的架构，操作和主要特性。

### 2.1 架构
