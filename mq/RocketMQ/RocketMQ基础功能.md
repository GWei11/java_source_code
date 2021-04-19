# 发送与消费

生产者默认使用的类是 DefaultMQProducer

## 发送类型

### 同步发送

* 同步发送指的是发送消息给 Broker 之后需要等待结果
* DefaultMQProducer 类中的如下方法就是同步发送方法

```java
public SendResult send(Message msg) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    return this.defaultMQProducerImpl.send(msg);
}
```

### 异步发送

* 一般的异步发送都是使用回调机制，对于发送端而言，消息发送之后就立即返回，发送端不会阻塞，等服务端收到消息之后会调用发送端的回调方法来告知发送端消息是发送成功还是失败。
* DefaultMQProducer 类中的如下方法就是异步发送方法

```java
public void send(Message msg, SendCallback sendCallback) throws MQClientException, RemotingException, InterruptedException {
    this.defaultMQProducerImpl.send(msg, sendCallback);
}

// SendCallback
public interface SendCallback {
    void onSuccess(SendResult var1);
    void onException(Throwable var1);
}
```

### Oneway发送

* 在一些对速度要求高，但是可靠性要求不是很高的场景下（比如日志收集），可以采用 Oneway 的方式来进行发送。
* Oneway 方式只发送请求而不需要等待应答，即 **数据写入客户端的 Socket 缓冲区就返回**，不需要等待对方返回结果。

* DefaultMQProducer 类中的如下方法就是异步发送方法

```java
public void sendOneway(Message msg) throws MQClientException, RemotingException, InterruptedException {
    this.defaultMQProducerImpl.sendOneway(msg);
}
```

### 延迟发送 TODO



### 发送事务消息 TODO



## 消费的方式

对于消费者来说有两种消费方式，一种是 pull 模式，另外一种是 push 模式。

* pull 模式就是消费者主动的从 Broker 中获取数据，使用 pull 模式相关代码如下

```java
public class MyConsumer {

    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        // 消息的拉取
        DefaultMQPullConsumer pullConsumer = new DefaultMQPullConsumer();
        // 表示获取一个主题下面的队列， 一个主题对应多个队列
        final Set<MessageQueue> messageQueues = pullConsumer.fetchSubscribeMessageQueues("tp_demo_05");
        // 循环队列，获取消息，一个队列对应多个消息
        for (MessageQueue messageQueue : messageQueues) {
            // 指定消息队列，指定标签过滤的表达式，消息偏移量和单次最大拉取的消息个数
            pullConsumer.pull(messageQueue, "TagA||TagB", 0L, 10);
        }
        // 启动消费者
        pushConsumer.start();
    }
}
```



* push 模式就是 Broker 主动的将数据推送给订阅了相关主题的消费者, 使用 push 方式相关代码如下

```java
public class MyConsumer {

    public static void main(String[] args) throws MQClientException, RemotingException, InterruptedException, MQBrokerException {
        // 消息的推送
        DefaultMQPushConsumer pushConsumer = new DefaultMQPushConsumer();
		// 对于消息的推送而言有一个订阅的步骤，只有告诉 Broker 我需要哪种类型（对应Topic）的消息，Broker才能推给我
        // subExpression表示对标签的过滤：比如TagA||TagB|| TagC 表示这三种类型标签都可以    *表示不对消息进行标签过滤
        pushConsumer.subscribe("tp_demo_05", "*");
        // 订阅完成之后需要添加消息监听器，一旦有消息推送过来，就进行消费
        pushConsumer.setMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                final MessageQueue messageQueue = context.getMessageQueue();
                System.out.println(messageQueue);
                for (MessageExt msg :msgs) {
                    try {
                        System.out.println(new String(msg.getBody(), "utf-8"));
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }
                // 消息消费成功
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        pushConsumer.start();
    }
}
```



## 消费的模式

### 集群模式

* 集群模式就是一个 ConsumerGroup 里面的 Consumer 实例会平摊消费消息，也就是每一个 Consumer 实例消费部分，这个组里面的所有 Consumer 实例合起来才是消费的全部。

#### maven 依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.7.1</version>
</dependency>
```

#### 消费者测试代码

```java
package com.example.demo.consumer;

import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyContext;
import org.apache.rocketmq.client.consumer.listener.ConsumeConcurrentlyStatus;
import org.apache.rocketmq.client.consumer.listener.MessageListenerConcurrently;
import org.apache.rocketmq.common.consumer.ConsumeFromWhere;
import org.apache.rocketmq.common.message.MessageExt;
import org.apache.rocketmq.common.message.MessageQueue;
import org.apache.rocketmq.common.protocol.heartbeat.MessageModel;

import java.io.UnsupportedEncodingException;
import java.util.List;
import java.util.Random;

public class MyConsumerTwo {
    public static void main(String[] args) throws Exception {
        // 消息的推送
        DefaultMQPushConsumer pushConsumer = new DefaultMQPushConsumer("consumer-demo");
        pushConsumer.setNamesrvAddr("开启了RocketMq的机器ip:9876");
        //pushConsumer.setMessageModel(MessageModel.BROADCASTING);
        // 这里使用的是集群模式
        pushConsumer.setMessageModel(MessageModel.CLUSTERING);
        pushConsumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);
        // 这里是设置不同的实例，做个标记，如果打开了这段代码，那么启动该类多次的时候，不能达到集群的效果，多个消费者之间消费的消息是一样的
        //pushConsumer.setInstanceName("instance" + new Random(10).nextInt());
        // 对于消息的推送而言有一个订阅的步骤，只有告诉 Broker 我需要哪种类型（对应Topic）的消息，Broker才能推给我
        // subExpression表示对标签的过滤：比如TagA||TagB|| TagC 表示这三种类型标签都可以    *表示不对消息进行标签过滤
        pushConsumer.subscribe("tp_demo_05", "*");
        // 订阅完成之后需要添加消息监听器，一旦有消息推送过来，就进行消费
        pushConsumer.setMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                final MessageQueue messageQueue = context.getMessageQueue();
                System.out.println(messageQueue);
                for (MessageExt msg :msgs) {
                    try {
                        System.out.println(new String(msg.getBody(), "utf-8"));
                    } catch (UnsupportedEncodingException e) {
                        e.printStackTrace();
                    }
                }
                // 消息消费成功
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        pushConsumer.start();
    }
}
```

#### 生产者测试代码

```java
package com.example.demo.producer;

import org.apache.rocketmq.client.exception.MQClientException;
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendCallback;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import org.apache.rocketmq.remoting.common.RemotingHelper;

/**
 * @Classname MyProducer
 * @Description 异步的Producer
 * @Date 2021/4/8 5:50
 * @Created by 咖啡杯里的茶
 */
public class MyProducer {
    /**
     * 异步发送
     *
     * @throws Exception
     */
    private static void asyncProducer() throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer();
        producer.setProducerGroup("group_demo_one1");
        producer.setNamesrvAddr("开启了RocketMq的机器ip:9876");
        producer.start();
        for (int i = 0; i < 10; i++) {
            Message message = new Message("tp_demo_05", "TagA1",
                    ("消息测试" + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            producer.send(message, new SendCallback() {
                @Override
                public void onSuccess(SendResult sendResult) {
                    System.out.println("发送成功" + sendResult);
                }

                @Override
                public void onException(Throwable e) {
                    System.out.println("发送失败" + e.getMessage());
                }
            });
        }
        Thread.sleep(10000);
        producer.shutdown();
    }
}

```

#### 测试结果

生产者运行结果

![](https://gitee.com/GWei11/picture/raw/master/20210410053140.png)

两个消费者运行的结果

![](https://gitee.com/GWei11/picture/raw/master/20210410053232.png)

![](https://gitee.com/GWei11/picture/raw/master/20210410053258.png)

#### 遗留问题

**如果上面设置实例的那行代码被打开了，那么起不到集群的效果，也就是多个实例并不是均摊所有消息，而是多个实例还是消费相同的消息，为什么？**

```java
pushConsumer.setInstanceName("instance" + new Random(10).nextInt());
```



#### 如何在 idea 中启动一个类多次

比如上面的 MyConsumerTwo 这个类想要启动多次，来模拟多个消费者，在 idea 中的操作步骤如下

1. 先启动 MyConsumerTwo 这个类
2. 启动之后编辑该类的配置信息

![](https://gitee.com/GWei11/picture/raw/master/20210410051927.png)

3. 在配置信息中勾选 Allow parallel run 选项

![](https://gitee.com/GWei11/picture/raw/master/20210410052008.png)

4. 做完上述操作之后可以继续启动 MyConsumerTwo 这个类

### 广播模式

* 无论是广播模式还是集群模式，谈论的前提是 **组** 这个概念
* 广播模式就是同一个组里面的不同消费者实例没有差别，每一个消费者都会消费订阅的主题下面的所有消息

#### 测试代码

就是将上面的 MyCustomerTwo 类中设置消息模式的地方改成如下所示即可。

```java
// 这里使用的是广播模式
pushConsumer.setMessageModel(MessageModel.BROADCASTING);
//pushConsumer.setMessageModel(MessageModel.CLUSTERING);
```

#### 测试结果

![](https://gitee.com/GWei11/picture/raw/master/20210410054121.png)

# 消息的存储 TODO





# 消息的过滤

## 什么是消息过滤

RocketMQ 中消费者是根据 Topic 来进行消息的消费的，但是 Topic 是一个比较大的维度，有的消费者不想要 Topic 下某些消息，那就需要对自己想要的那部分消息消息进行标记（生产者那端进行标记），然后消费者消费的时候才可以根据这些标记来拿到自己想要的那部分消息即可，这就是消息过滤。

## 消息过滤的方式

RocketMQ 中支持两种消息过滤的方式

* 使用 Tags 标签来进行过滤
* 使用 SQL92 语法来进行过滤

### 使用 Tags 来进行过滤

先来看一下发送者发送消息的代码

```java
public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer();
        producer.setProducerGroup("group_demo_one1");
        producer.setNamesrvAddr("启动了RocketMQ的机器ip:9876");
        producer.start();
        for (int i = 0; i < 10; i++) {
            // 重点看创建消息的过程
            Message message = new Message("TopicTest1", "TagA1", ("消息" + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
            SendResult sendResult = producer.send(message);
            System.out.println("发送结果：" + sendResult.toString());
        }
        producer.shutdown();
    }
```

在上面的代码中可以看一下 Message 的构造方法

```java
public Message(String topic, String tags, byte[] body) {
    this(topic, tags, "", 0, body, true);
}
```

1. 第一个参数是 Topic
2. 第二个参数是 tags
3. 第三个参数是消息主体

再看一下消费者端如何根据 tags 来进行过滤

在 pull 模式下看下面的代码：

```java
/**
 * @Classname MyConsumer
 * @Description 消费者
 * @Date 2021/4/8 5:50
 * @Created by 咖啡杯里的茶
 */
public class MyConsumer {

    public static void main(String[] args) throws Exception {
        myPullConsumer();
    }
    
    private static void myPullConsumer() throws Exception {
        DefaultMQPullConsumer consumer = new DefaultMQPullConsumer("consumer_grp_1");
        consumer.setNamesrvAddr("启动了RocketMQ的机器ip:9876");
        consumer.start();
        // 获取指定主题的消息， 一个Topic下面有多个MessageQueue
        Set<MessageQueue> messageQueues = consumer.fetchSubscribeMessageQueues("TopicTest1");
        // 遍历该主题的各个消息队列，进行消息的消费
        for (MessageQueue messageQueue : messageQueues) {
            /**
             *  第一个参数是MessageQueue对象，代表当前主题的一个消息队列
             *  第二个参数是一个表达式，对接收的消息按照tag进行过滤，支持 “tag1 || tag2 || tag3” 或者 “*” 类型的方法， * 表示不对消息进行过滤
             *  第三个参数是偏移量，表示消息从这里开始消费
             *  第四个参数表示每次最多拉取多个条消息
             */
            PullResult result = consumer.pull(messageQueue, "*", 0, 10);
            System.out.println("message*****queue****" + messageQueue);
            // 获取从指定消息队列中拉取的消息， 一个MessageQueue下面有多个Message
            List<MessageExt> msgFoundList = result.getMsgFoundList();
            if (msgFoundList == null) {
                continue;
            }
            for (MessageExt messageExt :
                    msgFoundList) {
                System.out.println(messageExt);
                System.out.println(new String(messageExt.getBody(), "utf-8"));
            }

        }
        // 关闭消费者
        consumer.shutdown();
    }
}
```

重点看 consumer.pull() 这个方法以及上面 的注释

对于 push 模式下就是看 consumer.subscribe消息订阅这个方法的代码

```java
// 第二个参数就是传递的 tags 过滤条件
public void subscribe(String topic, String subExpression) throws 
    MQClientException {
        this.defaultMQPushConsumerImpl.subscribe(this.withNamespace(topic), subExpression);
    }
```

### 使用 SQL92 来进行过滤

这是一种使用 sql 语句来查询的方式来进行过滤，说这种形式之前，先说一下自定义属性

#### 自定义属性

tags 标签就是 MessageQueue 的一个属性，只不过这个属性是系统提供给我们的，现在我们可以自己来添加属性（生产者一端添加属性）

```java
Message message = new Message("tp_demo_05", "TagA1",
                    ("消息测试" + i).getBytes(RemotingHelper.DEFAULT_CHARSET));
// 对消息添加自定义属性
message.putUserProperty("customkey", "value");
producer.send(message);
```

再看消息着一端， 看订阅方法

```java
public void subscribe(String topic, MessageSelector messageSelector) throws MQClientException {
        this.defaultMQPushConsumerImpl.subscribe(this.withNamespace(topic), messageSelector);
    }
```

```java
public class MessageSelector {
    private String type;
    private String expression;

    private MessageSelector(String type, String expression) {
        this.type = type;
        this.expression = expression;
    }

    public static MessageSelector bySql(String sql) {
        return new MessageSelector("SQL92", sql);
    }

    public static MessageSelector byTag(String tag) {
        return new MessageSelector("TAG", tag);
    }

    public String getExpressionType() {
        return this.type;
    }

    public String getExpression() {
        return this.expression;
    }
}
```

可以看到第二个参数 MessageSelector 有两种形式，一种是使用 Tag， 一种是使用 sql，就拿我们上面的自定义的属性 customkey 来说，可以这样写消费者代码

```java
consumer.subscribe("tp_demo_05", MessageSelector.bySql("customkey = '2'"));
```

#### SQL92基本语法

1. 数字比较： >, >= , < , <=, BETWEEN, =
2. 字符串比较： =，<>, IN, IS NULL 或者 IS NOT NULL
3. 逻辑比较： AND， OR, NOT
4. 对于字符串需要使用单引号引起来

#### 让 SQL92 语法格式生效

要想让这种形式生效，还有两个步骤需要完成

1. 修改 conf/broker.conf 这个配置文件，在最后面加上下面的代码

```shell
enablePropertyFilter=true
```

2. 在启动 broker 的时候需要带上这个配置文件

```shell
nohup sh ./bin/mqbroker -n 部署了RocketMQ的机器ip:9876 -c conf/broker.conf autoCreateTopicEnable=true &
```

