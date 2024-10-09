# fast DDS

## ch01 开始

### 1.1 什么是DDS

数据分发服务(Data Distribution Service,DDS)是为分布式软件通信提供的一种以数据为中心的通信协议。该协议描述了支持数据提供者(data providers)和数据消费者(data consumers)之间通信的通信接口（APIs）以及通信语义(Communication Semantics)

因为DDS是一个以数据为中心的发布订阅模型(Data-Centric Publish Subscribe model,aka DCPS),所以它的实现中有三个应用实体：

+ 发布实体：负责定义信息生成对象以及属性
+ 订阅实体：负责定义信息消费对象以及属性
+ 配置实体：定义作为话题传输的消息类型，以及根据消息质量(Quality of Service,aka QoS)创建发布者和订阅者，并确保以上实体有正确的性能（即创建发布者和订阅者，并提供可靠的QoS）

DDS使用QoS来定义DDS实体的行为特征。QoS由独立的QoS策略（由QoSPolicy派生出来的类型对象）组成。这些内容在3.1.2策略章节描述