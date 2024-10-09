# mosquitto

> TODO：设计模式完善，该实现使用单例模式，整个应用程序只能有一个mqttclient

简单封装

`MqttClient.h`

```c++
/*
 * @Author: OrangeHan
 * @Date: 2024-09-25
 * @LastEditors: OrangeHan
 * @LastEditTime: 2024-09-26
 * @FilePath: /deploy/include/tools/MqttClient.h
 * @Description: 
 * 
 * Copyright (c) 2024 by OrangeHan, All Rights Reserved. 
 */
#pragma once

//standard cpp
#include <string>
#include <string_view>
#include <functional>
#include <iostream>
//third party
#include "mosquitto.h"
#include "mosquittopp.h"
#include "spdlog/spdlog.h"


//alias
// using MqttConnectCallback = std::function<void(struct mosquitto*,void *,int)>;
using MqttMsgCallback = std::function<void(const std::string_view)>;
// using MqttDisconnectCallback = std::function<void(struct mosquitto*,void*,int)>;
class MqttClient;

//暂时使用单例模式
//fix me:添加注册以及解绑函数，即，增加统一程序对多个mqtt客户端的支持
class Factory{
public:
    static MqttClient* Instance(std::string_view ip,int port);
    static std::string ip_str_;
    static int port_;
private:
    Factory() = default;
    static MqttClient* instance_;
    
};


class MqttClient{
public:
    explicit MqttClient(std::string_view ip,int port);
    MqttClient(const MqttClient&) = delete;
    void operator=(const MqttClient&) = delete;

    bool initAndLoop();

    bool stopLoop(bool force = true);

    ~MqttClient();

    bool pubTopic(std::string_view topic_name,std::string_view msg,int qos = 0,bool retain = false,int* msg_id = nullptr);
    bool subTopic(std::string_view topic_name,int qos = 0,int* msg_id = nullptr);
    // void setConnectionCallback(MqttConnectCallback&& cb);
    void setMsgCallback(MqttMsgCallback&& cb){
        msg_cb_ = std::move(cb);
    }
    // void setDisconnectionCallback(MqttDisconnectCallback&& cb);
    
    void handleMsg(std::string_view);

private:
    static void connectionCallback(struct mosquitto *mosq, void *obj, int rc);
    static void msgCallback(struct mosquitto *mosq, void *obj, const struct mosquitto_message *msg);
    static void disconnectCallback(struct mosquitto*,void* obj,int rc);

private:
    struct mosquitto* mosq_client_;
    std::string ip_str_;
    int port_;
    bool is_valid_{false};
    bool loop_started_{false};
    // MqttConnectCallback conn_cb_;
    MqttMsgCallback msg_cb_;
    // MqttDisconnectCallback disconn_cb_;
};
```

`MqttClinet.cc`

```c++
/*
 * @Author: OrangeHan
 * @Date: 2024-09-25
 * @LastEditors: OrangeHan
 * @LastEditTime: 2024-09-26
 * @FilePath: /deploy/src/tools/MqttClient.cpp
 * @Description: 
 * 
 * Copyright (c) 2024 by OrangeHan, All Rights Reserved. 
 */
#include "mosquitto.h"

#include "MqttClient.h"

MqttClient* Factory::instance_{nullptr};
std::string Factory::ip_str_;
int Factory::port_;

MqttClient* Factory::Instance(std::string_view ip,int port){
    if(instance_ == nullptr){
        Factory::ip_str_ = ip.data();
        Factory::port_ = port;
        instance_ = new MqttClient(ip,port);
    }
    return instance_;
}



MqttClient::MqttClient(std::string_view ip,int port)
:ip_str_(ip)
,port_(port)
{
}

/**
 * @description: 三方库要求，在使用前必须执行mosquitto_lib_init,用户使用前必须手动调用该函数
 * @return {*}
 */
bool MqttClient::initAndLoop(){
    //初始化库
    int res = mosquitto_lib_init();
    if(res != MOSQ_ERR_SUCCESS){
        SPDLOG_ERROR("mosquitto lib init failed"); 
        mosquitto_lib_cleanup();
        return false;
    }
    //创建核心结构体（所有操作仅基于该结构体）
    mosq_client_ = mosquitto_new(nullptr,true,nullptr);
    if(mosq_client_ == nullptr){
        SPDLOG_ERROR("create mosquitte client error");
        mosquitto_lib_cleanup();
        return false;
    }
    //设置连接以及接受消息的回调函数
    mosquitto_connect_callback_set(mosq_client_,connectionCallback);
    mosquitto_disconnect_callback_set(mosq_client_,disconnectCallback);
    mosquitto_message_callback_set(mosq_client_,msgCallback);

    //正式连接服务器
    int keepalive = 5;//心跳间隔
    if(mosquitto_connect(mosq_client_,ip_str_.c_str(),port_,keepalive) != MOSQ_ERR_SUCCESS){
        SPDLOG_ERROR("connect 2 server failed");
        mosquitto_destroy(mosq_client_);
        mosquitto_lib_cleanup();
        return false;
    }

    if(mosquitto_loop_start(mosq_client_) != MOSQ_ERR_SUCCESS){
        SPDLOG_ERROR("start loop failed");
        mosquitto_destroy(mosq_client_);
        mosquitto_lib_cleanup();
        return false;
    }
    SPDLOG_INFO("MqttClient established");
    loop_started_ = true;
    is_valid_ = true;
    return true;
}

MqttClient::~MqttClient(){
    if(is_valid_){
        stopLoop();
        mosquitto_destroy(mosq_client_);
        mosquitto_lib_cleanup();
    }
}

bool MqttClient::pubTopic(std::string_view topic_name,std::string_view msg,int qos,bool retain,int* msg_id){
    int ret = mosquitto_publish(mosq_client_,msg_id,topic_name.data(),msg.length(),msg.data(),qos,retain);
    if(ret != MOSQ_ERR_SUCCESS){
        SPDLOG_ERROR("publish error:{}",mosquitto_strerror(ret));
        return false;
    }
    return true;
}

bool MqttClient::subTopic(std::string_view topic_name,int qos,int* msg_id){
    int ret = mosquitto_subscribe(mosq_client_,msg_id,topic_name.data(),qos);
    if(ret != MOSQ_ERR_SUCCESS){
        SPDLOG_ERROR("subscribe error:{}",mosquitto_strerror(ret));
        return false;
    }
    return true;
}


void MqttClient::handleMsg(std::string_view msg){
    if(msg_cb_){
        msg_cb_(msg);
    }
}

bool MqttClient::stopLoop(bool force){
    if(loop_started_){
        int ret = mosquitto_loop_stop(mosq_client_,force);
        if(ret != MOSQ_ERR_SUCCESS){
            SPDLOG_ERROR("loop stop failed:{}",mosquitto_strerror(ret));
            return false;
        }
        loop_started_ = false;
        return true;
    }else{
        SPDLOG_INFO("loop hasn't been started,no need to stop");
        return true;
    }
}


//static memeber function
void MqttClient::connectionCallback(struct mosquitto *mosq, void *obj, int rc){
    if(rc == 0){
        SPDLOG_INFO("mqtt client connect success");
    }
}

void MqttClient::msgCallback(struct mosquitto *mosq, void *obj, const struct mosquitto_message *msg){
    SPDLOG_INFO("received msg:{}",reinterpret_cast<const char*>(msg->payload));
    MqttClient* instance = Factory::Instance(Factory::ip_str_,Factory::port_);
    instance->handleMsg(reinterpret_cast<const char*>(msg->payload));
}


void MqttClient::disconnectCallback(struct mosquitto* mosq,void* obj,int rc){
    SPDLOG_INFO("disconnected");
}
```

