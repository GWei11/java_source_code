

# 安装 RocketMQ

* 使用 wget 命令将 RocketMQ 下载下来

```shell
wget https://archive.apache.org/dist/rocketmq/4.5.1/rocketmq-all4.5.1-bin-release.zip
```

当然也是可以使用其他的版本，进入 [apach官网](https://archive.apache.org/dist/) ，然后找到 RocketMQ 对应的版本的地址即可。

* 将下载下来的文件解压到 `/opt` 目录中

  ![](https://gitee.com/GWei11/picture/raw/master/20210406221638.png)

```shell
unzip xxxx -d /opt
```

这里 RocketMQ 是使用 zip 形式进行压缩的，所以需要 unzip 来进行解压，如果提示没有该命令的话需要使用下面的命令来进行安装。

```shell
yum install zip unzip
```

* 解压完成之后修改 `/etc/profile` 文件的内容，如下图所示，在该文件的末尾添加如下内容。
  ![](https://gitee.com/GWei11/picture/raw/master/20210406222108.png)
* 修改完该配置文件之后，可能不会立即生效，所以最好使用 source 命令来更新一下该文件。

```shell
source /etc/profile
```



# 安装 JDK

* 到 [Oracle官网](https://www.oracle.com/java/technologies/javase-downloads.html#JDK16) 去下载 jdk ，比如我这里下载的是 `jdk-11.0.5-linux-x64.rpm` ( Linux版本的 )。
* 可以使用 Xshell 连接远程的 Linux 服务器，然后使用 rz 命令来将下载的 jdk 上传到服务器中。
* 使用下面的命令来进行安装

```
rpm -ivh jdk-11.0.5-linux-x64.rpm
```

* 配置环境变量（就像在 Window 系统中配置 JAVA_HOME 环境变量一样）

```shell
vim /etc/profile
```

在最后面添加如下代码

```shell
export JAVA_HOME=/usr/java/jdk-11.0.5
export CLASSPATH=.:$JAVA_HOME/lib/tools.jar:$JAVA_HOME/jre/lib/dt.jar:$JAVA_HOME/lib
export PATH=$PATH:$JAVA_HOME/bin:$JAVA_HOME/jre/bin
```

![](https://gitee.com/GWei11/picture/raw/master/20210406223313.png)

* 检查是否安装完成

```shell
java -version
```

* rpm 卸载 jdk

```shell
rpm -e jdk的版本
```



# 运行与关闭 RocketMQ 的操作

## 单机版

启动单机版的 RocketMQ 消息队列服务步骤如下：

进入到 解压的 RocketMQ 的 bin 目录，然后运行

```shell
# 启动 NameServer 命令
mqnamesrv
# 查看启动日志
tail -f ~/Logs/rocketmqlogs/namesrv.log
```

**错误问题：**

### namesrv 修改

可能在执行 mqnamesrv 命令的时候就会出现下面的问题：

```java
Java HotSpot(TM) 64-Bit Server VM warning: Option UseConcMarkSweepGC was deprecated in version 9.0 and will likely be removed in a future release.
Unrecognized VM option 'UseCMSCompactAtFullCollection'
```

我们运行的是 mqnamesrv 这个命令，现在我们可以使用 cat 命令来查看一下这个文件里面的具体内容：

```shell
cat mqnamesrv
```

在输出的结果中最后一行可以看到如下内容：

```shell
sh ${ROCKETMQ_HOME}/bin/runserver.sh org.apache.rocketmq.namesrv.NamesrvStartup $@
```

也就是我们运行 mqnamesrv 的时候最终是运行的 runserver.sh 这个 shell 脚本，现在再使用 cat 命令查看一下这个文件

```shell
cat runserver.sh
```

部分内容如下：

```java
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
JAVA_OPT="${JAVA_OPT} -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8  -XX:-UseParNewGC"
```

在我们启动时遇到的错误问题显示是不能识别 UseCMSCompactAtFullCollection 这个参数，在上面的输出结果中果然看到了这个参数，这是因为这个参数在 jdk8 里面有，但是到了 jdk11 就没有这个参数了，所以解决方案有两个

1. 不使用 jdk11， 使用 jdk8
2. 在启动脚本里面去除掉这些参数
   ![](https://gitee.com/GWei11/picture/raw/master/20210407053518.png)

具体需要删除的几个值如下：

```shell
UseCMSCompactAtFullCollection
UseParNewGC
UseConcMarkSweepGC
# 下面这里是修改内容，在 runserver.sh脚本中给的是 4g，我们服务器没有没有这么大内存，所以改小一点
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -
XX:MetaspaceSize=64mm -XX:MaxMetaspaceSize=160mm"
```

重新运行下面命令

```shell
mqnamesrv
```

然后又出现了问题，提示没有 `-Xloggc` 这个参数

再看 runserver.sh 这个脚本的内容发现有如下代码

```shell
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:/dev/shm/rmq_srv_gc.log -XX:+PrintGCDetails"
```

所以我们继续安装提示来修改即可，将 `-Xlogcc` 修改为 `-Xlog:gc`

再启动的话会提示没有没有主类的错误

```java
Error: Unable to initialize main class org.apache.rocketmq.namesrv.NamesrvStartup
Caused by: java.lang.NoClassDefFoundError: org/apache/rocketmq/srvutil/ShutdownHookThread
```

这个是需要注释掉 CLASSPATH（如何修改看下面完整文件即可）, 改完之后的 runserver.sh 如下：

```sh
error_exit ()
{   
    echo "ERROR: $1 !!"
    exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
#export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}
export CLASSPATH=${BASE_DIR}/lib/rocketmq-namesrv-4.5.1.jar:${BASE_DIR}/lib/*:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=64m -XX:MaxMetaspaceSize=160m"
JAVA_OPT="${JAVA_OPT} -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8 "
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xlog:gc:/dev/shm/rmq_srv_gc.log -XX:+PrintGCDetails"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT}  -XX:-UseLargePages"
#JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

$JAVA ${JAVA_OPT} $@
```



### broker文件修改

在 bin 目录中输入如下命令进行修改

```shell
vim runbroker.sh
```

修改完成之后的文件如下：

```sh
export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
#export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}
# JVM Configuration
#JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xlog:gc:/dev/shm/mq_gc_%p.log"
#JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:+AlwaysPreTouch"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=15g"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages -XX:-UseBiasedLocking"
#JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

numactl --interleave=all pwd > /dev/null 2>&1
if [ $? -eq 0 ]
then
        if [ -z "$RMQ_NUMA_NODE" ] ; then
                numactl --interleave=all $JAVA ${JAVA_OPT} $@
        else
                numactl --cpunodebind=$RMQ_NUMA_NODE --membind=$RMQ_NUMA_NODE $JAVA ${JAVA_OPT} $@
        fi
else
        $JAVA ${JAVA_OPT} --add-exports=java.base/jdk.internal.ref=ALL-UNNAMED $@
fi
```

有一个地方是新加了代码，其余的只是修改，添加的地方是最后一行

```shell
$JAVA ${JAVA_OPT} --add-exports=java.base/jdk.internal.ref=ALL-UNNAMED $@
```

原来的内容是

```shell
$JAVA ${JAVA_OPT} $@
```

### tools.sh 文件修改

只需要删除一句代码

```shell
# 注释 JAVA_OPT="${JAVA_OPT} -
Djava.ext.dirs=${BASE_DIR}/lib:${JAVA_HOME}/jre/lib/ext"
```



### 启动 mqnamesrv

可以直接输入下面命令 (**当前目录是在 bin 目录**)

```shell
mqnamesrv
```

也可以使用

```shell
nohup sh mqnamesrv &  # 最后面的 & 符号表示后台启动
```

查看 mqnamesrv 的启动状态

```shell
ps -ef | grep mq
```

当然也可以看 namesrv 的启动日志，日志文件在用户的 家目录下面的 logs 下面。具体路径如下：

```shell
~/logs/rocketmqlogs/namesrv.log   # ~ 表示用户的家目录
```



### 启动 broker

可以直接输入下面命令(**当前目录在 bin 目录**)

```
mqbroker -n localhost:9876
```

也可以使用

```shell
nohup sh mqbroker -n localhost:9876 &  # 最后面的 & 符号表示后台启动
```

