# 下载并安装 Elasticsearch

## 卸载自带的jdk

* 查询系统是否安装了 jdk

```shell
java -version
```

* 如果查询出来的结果是安装了，接下来就是查询具体安装了哪些包

```shell
rpm -qa | grep java
```

![](https://gitee.com/GWei11/picture/raw/master/20210410212458.png)

* 根据上一步查询的结果卸载包

```shell
rpm -e --nodeps [jdk名称]
```

对于 noarch 文件是可以不用删除的。

## 安装jdk

* 到官网下载 jdk11 https://www.oracle.com/java/technologies/javase-downloads.html
* 将文件上传到 Centos7 系统中，并且解压压缩包（我使用的 jdk 是11版本的）

```shell
tar -zxvf jdk-11.0.10_linux-x64_bin.tar.gz 
```

* 移动文件到安装目录

```
mv jdk-11.0.10 /usr/java
```

* 配置 jdk 环境变量

```shell
vim /etc/profile
```

在 profile 文件的结尾添加如下内容：

```shell
JAVA_HOME=/usr/java
JRE_HOME=/usr/java/jre
CLASS_PATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
export JAVA_HOME JRE_HOME CLASS_PATH PATH
```

上面的 JAVA_HOME=/usr/java 是因为我将 jdk 解压后放在 /usr/java 目录里面了，如果你的不是在这个目录，更换为自己的目录即可。

* 让配置文件的修改生效

```shell
source /etc/profile
```

* 检查 jdk 是否安装成功

```shell
java -version
```

## 安装Elasticsearch

* 进入官网进行下载 https://www.elastic.co/cn/downloads/elasticsearch
* 选择 Linux 版本的
* 将下载的文件拷贝到 Centos 系统中
* 解压下载的 Elasticsearch 

```shell
tar -zxvf elasticsearch-7.3.0-linux-x86_64.tar.gz
```

* 移动解压后的文件到 usr 目录中

```shell
mv /root/elasticsearch-7.3.0 /usr/elasticsearch/
```

* 配置 Elasticsearch

```shell
vim /usr/elasticsearch/config/elasticsearch.yml
```

主要是取消几个注释，然后将集群环境里面的机器删除 node-2【注意每一个冒号后面有一个空格】

```shell
node.name: node-1
# 可以换成自己的ip，或者写成0.0.0.0表示所有ip
network.host: 0.0.0.0
http.port: 9200
# 默认这里写的是 node-1，node-2. 由于这里是单机版，上面的 node.name 就是 node-1，所以写一个 node-1 即可。
cluster.initial_master_nodes: ["node-1"]
```

* 按照实际需求修改内存设置

```shell
vim /usr/elasticsearch/config/jvm.options
```

根据实际情况修改内存，默认是 1G, 如果给的太小，可能会报内存不足的错误。这里我就使用默认值。

* 添加一个用户（比如我新增用户 lg），因为 Elasticsearch 默认使用 root 用户是无法启动的，需要修改为其他用户

```shell
useradd lg
passwd lg  # 这里是输入 lg 用户的密码
```

* 修改前面的 elasticsearch 目录的拥有者为我们新创建的 lg 用户

```shell
chown -R lg /usr/elasticsearch/
```

修改完成之后，可以使用下面的命令来查看结果

```shell
ll /usr   # 前面是小写的 LL
```

![](https://gitee.com/GWei11/picture/raw/master/20210411064105.png)

* 再修改 sysctl.conf 配置文件

```shell
vim /etc/sysctl.conf
```

在最后面添加下面的一段内容

```shell
vm.max_map_count=655360
```

然后执行下面命令让修改生效

```shell
sysctl -p
```

* 修改 limits.congp 配置文件

```shell
vim /etc/security/limits.conf
```

在最后面添加如下内容【内容很多，使用 vim 之后可以在命令模式下使用 G 跳转到最后一行】，注意前面的 * 也是需要添加的。

```shell
* soft nofile 65536
* hard nofile 65536
* soft nproc 4096
* hard nproc 4096
```

* 切换到刚刚创建的 lg 用户，启动 es

```shell
su lg # 切换用户
/usr/elasticsearch/bin/elasticsearch  # 启动 es
```

* 等待一段时间启动之后，在浏览器访问

```http
http://ip:9200
```

返回的结果是类似下面这样的 json 串就是启动成功了

```json
{
  "name" : "node-1",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "iGpe7O_kS32QsOr9bR5G_Q",
  "version" : {
    "number" : "7.3.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "de777fa",
    "build_date" : "2019-07-24T18:30:11.767338Z",
    "build_snapshot" : false,
    "lucene_version" : "8.1.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```



# 安装 kibana

es 是支持 restful 风格的 api 的，所以当启动 es 服务端之后，如果只是想测试的话可以直接使用 postman 来操作 es 服务端。

但是 es 也有很多配套的可视化的操作工具，比如 kibana，所以现在来安装一下 kibina

【小提示】：使用 xshell 连接 Centos 的时候，双击标签栏会新建一个窗口连接。

![](https://gitee.com/GWei11/picture/raw/master/20210411070638.png)

* 进入 elastic 官网 https://www.elastic.co/cn/ 可以在**产品** 目录下找到 kibina， 这里直接给一下下载地址 https://www.elastic.co/cn/downloads/

* 下载 Linux 版本的 kibina 并且上传到 Centos7 中
* 解压并改变 kibina 目录拥有者以及设置权限

```shell
tar -zxvf kibana-7.3.2-linux-x86_64.tar.gz  # 解压
mv /root/kibana-7.3.2-linux-x86_64 /usr/kibana    # 移动解压目录到 /usr/kibana 
chown -R lg /usr/kibana/  # 改变目录拥有者
chmod -R 777 /usr/kibana/   # 分配权限
```

* 修改配置文件

```shell
vim /usr/kibana/config/kibana.yml
```

主要是打开几个注释以及修改 elasticsearch.hosts 的访问地址

```shell
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.hosts: ["http://启动了es服务器的机器ip:9200"]
```

【小技巧】vim 中在**命令模式**下我们可以输入 `/server.port` 然后按下回车就会去搜索 server.port ,通过这种搜索可以比较快的找到对应位置。

* 配置成功之后切换用户

```shell
su lg
```

kibina 默认也是不使用root 用户登录的，如果想要使用 root 用户登录，需要在启动命令后面加上 `--allow-root `

```shell
/usr/kibana/bin/kibana   # 非root 用户登录的
```

在启动的过程中我遇到了下面的错误

```shell
Error: EACCES: permission denied, stat '*/translations/en.json'
```

这里可以看到是该文件没有授权，现在可以使用下面的命令来查找这个文件具体在什么地方

```shell
find / -name en.json
```

查找结果里面看到有四个位置，可以看一下，这几个文件是否真的没有 lg（就是你新建的那个用户） 用户的权限，因为我们上面其实已经授权了，所以可能你看到的结果是 lg 用户对这些文件都是有权限的，那就重新运行启动命令就好了，我的就是这样，再次运行的时候就自己好了。

* 访问 kibana

```http
http:启动了kibana机器的ip:5601
```

* kibana 的基础使用界面

![](https://gitee.com/GWei11/picture/raw/master/20210411073320.png)