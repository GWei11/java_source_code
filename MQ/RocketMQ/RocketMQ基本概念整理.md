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





# 为什么需要 ConsumerGroup

组的概念就类似我们平时去参加什么团体赛一样

* 首先比赛有很多种，比如数学竞赛，编程比赛等，这个种类就相当于 RocketMQ 里面的 Topic 的概念
* 现在假设我们有三个人合成一个组去参加比赛，那每一个组都有一个组号，你的组号和我的组号不一样，这个组号就相当于 RocketMQ 里面的 组（比如 ConsumerGroup）的概念
* 那这个组有什么用呢？你想啊，假如这个比赛一共有 90 题，我们有三个人参加，是不是可以分配一下每个人做个 30 题再说呢（当然能力强的可以多做一点，这个就设计到 负载均衡算法了，暂不管他）。我们同一个组的人拿一份试题就好了
  * 对应到 RocketMQ 中，就是每一个组都会拿到全量的消息，然后这些消息你们组内自己去随便分配吧

![](https://gitee.com/GWei11/picture/raw/master/20210423073528.png)