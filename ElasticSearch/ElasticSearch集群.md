# 单机环境搭建

[Centos7下安装Elasticsearch](http://mp.weixin.qq.com/s?__biz=Mzg3NDYwNjA4Ng==&mid=2247483727&idx=1&sn=84bf845f2c5404b0bbab2eab0a75807e&chksm=cecf7a26f9b8f3309e6eaef2e924a06d1129da3b0eff69a6c7d93f2b67710922c077126b768b&scene=21#wechat_redirect)



# 集群环境搭建

**【注意】：下面说的方法，是在 Centos7 中搭建的，其实也可以在 windows 中来搭建 Elasticsearch 集群，配置文件是一样的。**

## 拷贝文件（使用的 root 用户）

原来的 Elasticsearch 安装目录如下：

```shell
/usr/elasticsearch
```

现在搭建集群可以使用三台机器，在本地测试的话可以使用一台机器，然后单个节点的 port 不同。

我们将原来的 Elasticsearch 复制两份出来

```shell
cp /usr/elasticsearch/ /usr/elasticsearch1/ -rf    # -r 表示递归复制，即子文件夹也是可以复制的， -f 表示强制，如果有文件是不能打开的也可以复制
cp /usr/elasticsearch/ /usr/elasticsearch2/ -rf
```



## 修改配置文件

修改

```properties
cluster.name: my-es #集群名称 ---
node.name: node-1 # 节点名称
node.master: true #当前节点是否可以被选举为master节点，是：true、否：false ---
network.host: 0.0.0.0
http.port: 9200
transport.port: 9300 # ---
#初始化一个新的集群时需要此配置来选举master
cluster.initial_master_nodes: ["node-1","node-2","node-3"]
#写入候选主节点的设备地址 ---
discovery.seed_hosts: ["127.0.0.1:9300", "127.0.0.1:9301","127.0.0.1:9302"]
http.cors.enabled: true
http.cors.allow-origin: "*"
```

## 修改权限

前面几步的操作是在 root 用户下操作的，但是 Elasticsearch 是不能在 root 用户下启动的，所以还需要修改权限，我们创建另外一个用户 lg ， 将刚刚复制的几个文件夹的所属用户修改成 lg

```shell
chown -R lg /usr/elasticsearch1
chown -R lg /usr/elasticsearch2
```

## 删除 data 目录下的 node 数据

做完上述操作之后还需要 删除 data 目录下 node 节点的数据（因为之前可能启动过 ELasticsearch，所以已经会产生了一些数据）

当前所处目录是 Elasticsearch 安装目录

```shell
/usr/elasticsearch
```

在当前目录执行下面的命令删除 node 数据

```shell
m data/nodes/  -rf
```

我们在前面复制两份 Elasticsearch，名字叫做 Elasticsearch1 和 Elasticsearch2 ，所以这两个里面 的 node 节点数据也是需要删除的。



## 切换用户并启动 Elasticsearch

* 切换到 我们创建的 lg 用户
* 启动 ELasticsearch



# Cerebro 查看集群状态

搭建集群之后可以下载 Cerebro 来查看集群的状态。

[使用 Cerebro 查看集群状态](https://github.com/lmenezes/cerebro/releases)

该工具只是依赖 jvm ，所以只需要安装了 jdk 即可，然后直接启动 bin 目录下的 cerebro 命令即可。启动之后可以看到该工具监听的端口是多少，默认是 9000，所以可以访问 `http:ip:9000` 即可。在初始界面中我们输入 Elasticsearch 集群的任意一台机器 ip 和端口进行连接，就可以访问到我们的集群。

![](https://gitee.com/GWei11/picture/raw/master/20210426061506.png)

连接之后可以看到服务启动。

![](https://gitee.com/GWei11/picture/raw/master/20210426061724.png)

# 分片和副本

我们创建一个 index 的时候，其实是可以设置分片和副本的数量的，那么如何去理解分片和副本呢？

* 副本是比较好理解的，简单来说就是 “备胎“， 作用是实现高可用。
* 分片的作用是增加存储容量的，比如很多大楼都不止一部电梯，这栋大楼里面每一个分片就相当于是一个分片了，你想本来一个电梯只能坐18人，现在你又3个电梯，那一次性可以搭载的人 不是 18 * 3 了么。分片也是同样的道理，比如一个 index 下面有 90 个 document，如果没有分片，那么这个 index 下面的 90 个 document 是不是只能存储在 一个机器上， 这样就算搭建了集群也没有，但是现在如果分了6 片，然后集群中机器是 3 台， 那就可以每一台机器存储 30 个文档了。

![](https://gitee.com/GWei11/picture/raw/master/20210426070914.png)

既然有了分片，那么我们的数据时如何存储到对应分片上的呢，比如创建了一个文档，怎么知道该文档要放在哪一个分片上面去？

## 文档到分片的映射算法

### 目的

使文档可以均匀的分布在所有分片上，可以充分利用资源

### 算法

* 使用随机选择或者使用轮询机制
  * 虽然这两种方式都可以让文档比较均衡的分布在各个分片上面，但是查询的时候比较麻烦，还需要维护文档到分片的映射关系，否则只能查询所有的分片，显然不可取。
* es 通过下面的公式来**计算**（不是存储）文档到对应分片的映射关系

```shell
shard = hash(routing) % numer_of_primary_shards
```

* hash 算法可以保证将数据均衡的分布在分片中
* routing 是一个关键参数，默认是使用 文档id， 也可以自己指定。

### 问题

为什么新增机器后，Elasticsearch 不能充分利用新增机器的资源？

这是因为数据时存放在分片中的，而数据存放规则是根据 文档到对应分片的映射关系来计算出来的，也就是当我们在最开始创建索引的时候，就已经确定了分片，后面即使增加了机器，对于已经创建好的索引来说，分片是不会在在新机器中，所以之前已经创建好的 index 也就不能利用新的机器资源了。



# 集群的健康状态

从数据完整性的角度划分，集群健康状态分为三种

* green：所有的主分片和副分片都正常运行
* yellow：所有的主分片都正常执行，但不是所有的副分片都正常执行，这意味着存在单点故障风险
* red：有主分片没能正常运行