# RocketMQ 中的几个角色

![](https://gitee.com/GWei11/picture/raw/master/20210420052546.png)



* Producer: 消息的发送者
* Consumer：消息的接收者
* Broker：暂存和传输消息
  * Broker 分为 Master 和 Slave，一个 Master 可以对应多个 Slave，一个 Slave 只能对应一个 Master
* NameServer：用来管理 Broker 的，多台 NameServer 之间不交互
  * Broker 启动后，会和所有的 NameServer 保持长连接，定时发送心跳包，心跳包里面包含当前 Broker 信息（IP + 端口）以及存储所有的 Topic 信息，注册成功后，NameServer 集群就有 Topic 和 Broker 的映射关系。
* Topic：区分消息的种类，一个发送者可以发送消息给一个或多个 Topic，一个消息的接收者可以订阅一个或多个 Topic 消息
* Message Queue：相当于 Topic 的分区，用于并行发送和接收信息



# 什么是 MessageQueue 和 ConsumerQueue





