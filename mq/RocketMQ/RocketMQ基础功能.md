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

### 延迟发送



### 发送事务消息



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

### 广播模式

* 无论是广播模式还是集群模式，谈论的前提是 **组** 这个概念
* 广播模式就是同一个组里面的不同消费者实例没有差别，每一个消费者都会消费订阅的主题下面的所有消息



### 集群模式

* 集群模式就是一个 ConsumerGroup 里面的 Consumer 实例会平摊消费消息，也就是每一个 Consumer 实例消费部分，这个组里面的所有 Consumer 实例合起来才是消费的全部。









# 消息的存储

