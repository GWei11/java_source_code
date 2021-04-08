

# 启动 Producer 出现的问题

## Exception in thread "main" org.apache.rocketmq.remoting.exception.RemotingTooMuchRequestException: sendDefaultImpl call timeout

目前没有找到原因，但是在网上看了一篇文章可以解决我的问题。

参考文章 https://blog.csdn.net/qq_21460229/article/details/104351178 ， 主要步骤如下：

* 在启动 namesrv 过程中需要加上公网 ip 参数

```shell
nohup ./bin/mqnamesrv -n 公网ip:9876
```

* 修改 broker 对应的配置文件（当前目录在 RocketMQ 根目录中）

```shell
vim conf/broker.conf
```

 在最后面加上一段：

```shell
brokerIP1=公网 ip
```

* broker 启动的时候需要加上参数，具体命令如下

```shell
nohup sh bin/mqbroker -n 你的公网IP:9876 -c conf/broker.conf autoCreateTopicEnable=true &
```

