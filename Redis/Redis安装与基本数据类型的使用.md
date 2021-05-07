# 安装和启动 Redis

[redis 历史版本下载地址](http://download.redis.io/releases/)

## 安装 redis

* 到 [redis官网](https://redis.io/) 下载 Linux 版本的 redis
* 将下载的压缩包传入到 Linux 系统中，放置在 /opt 目录下
* 安装 c 语言编译环境

```shell
yum install centos-release-scl scl-utils-build
yum install -y devtoolset-8-toolchain
scl enable devtoolset-8 bash
```

* 测试 gcc 版本

```shell
gcc --version
```

* 到 /opt 目录下去执行如下解压命令来解压 redis 压缩文件

```shell
tar -zcvf redis-6.2.2.tar.gz
```

* 解压完成之后进入 redis-6.2.2 目录

```shell
cd redis-6.2.2
```

* 在该目录下面执行 make 命令

```shell
make
```

* 如果在之前没呀安装好 c 语言编译环境，在这一步会出现错误：没有 Jemalloc/jemalooc.h 这个文件， 解决方案是运行如下所示命令

```shell
make distclean
```

然后再一次执行 make 命令，当然如果没有这个问题，执行一次 make 就可以了。

* 跳过 make test 命令，直接执行 make install 来安装

```shell
make install
```

* 通过上面的步骤安装之后，redis 的安装在 /usr/local/bin 目录下面

## 启动和关闭 redis

### 前台启动的方式

* 启动服务端直接使用下面的 命令

```shell
redis-server
```

这种方式不推荐，因为当前窗口关闭的时候，redis 也会停止服务端

### 后台启动的方式

* 首先备份一下 redis.conf 这个配置文件，因为我们在后端启动的时候需要使用配置文件。redis 的安装目录在 /opt/redis-6.2.2 下面， 备份的命令如下所示：

```shell
cp /opt/redis-6.2.2/redis.conf /opt/redis-6.2.2/redis-bak.conf
```

* 修改 redis.conf 配置文件将 `daemonize no` 的配置修改成 `daemonize yes` 即可。
* 启动 redis 服务端（根据配置文件来启动）（可以在任意位置来输入如下命令）

```shell
redis-server /opt/redis-6.2.2/redis.conf
```

* 使用 ps 命令来查看 redis-server 是否启动

```shell
ps -ef | grep redis
```

通过上面命令的执行结果就可以看到是否后台是否在运行 redis-server 服务

* 使用 redis 客户端访问（可以在任意位置输入如下命令）

```shell
redis-cli
```

## redis 的几个可视化客户端工具

* Redis Desktop Manager [地址](https://rdm.dev/)， 现在收费
* [medis](http://getmedis.com/) , 免费
* [AnotherRedisDesktopManager](https://github.com/qishibo/AnotherRedisDesktopManager)

# Reis 的常用几种基本数据类型

redis 是 key-value 形式的存储系统，key 都是字符串，但是 value 是有多种类型的。下面列举出几种常用的 value 类型。

## string

* 在 redis 中 String 类型可以表达 3 种类型的值：字符串、整数、浮点数 【示例】：100.01 其实是一个 6 位的字符串。

### 常用命令

#### set key value 

直接给一个 key 赋值的命令，下面命令中 hello 是 key ， world 是value

```shell
set hello world
```

#### get key

 取值

```shell
get hello
```

#### getset key value  

作用是设置这个 key 新的值，返回的是这个 key 的旧值， 比如下面的命令中，hello 这个 key 对应的 value 是 ‘测试第二个值’， 但是之前的值是 ‘world’，所以这个命令会返回 ‘world’

```shell
getset hello 测试第二个值
```

#### setnx

 当value 不存在时就是赋值，如果 value 已经存在了，执行该命令后 value 的值不会改变。

```shell
setnx hello 测试setnx
```

上面命令执行完成之后，hello 这个key 对应的 value 还是 ‘测试第二个值’， 因为已经有了个 hello 这个 key，所以 setnx 命令就不会给这个 key 进行赋值。

#### append key value

向尾部添加值

```shell
get hello # 此时值为  '测试第二个值'
append hello 尾部的值
get hello # 此时值为    '测试第二个值尾部的值'
```

#### strlen key 

获取字符串长度

```shell
strlen hello   # 返回的是 30， ‘测试第二个值尾部的值’是10个字，30个字节
```

#### incr key 

递增数字（对应的 value 必须是 数字 类型的才可以）

```shell
set incr_key 1  # 设置 incr_key 的 value 是1
incr incr_key  # 将这个 key 递增1
get incr_key # 获取的结果是 2
```

#### incrby key increment

增加指定的整数

```shell
incrby incr_key 3  # 返回的结果是 5 ，因为上一步的结果是2 ，现在递增 3
```

#### decr key 

递减数字

```shell
decr incr_key  # 返回的结果是 4
```

#### decrby key decrement 

减少指定的整数

```shell
decrby incr_key 2 # 返回的结果是2 ，上一步执行完之后结果是哦4， 现在递减2， 所以结果是2
```

### 使用场景

* incr 方法常用于乐观锁， 因为 incr 是递增数字，而是该操作是原子性的操作。
* setnx 常用语分布式锁，当 value 不存在时才会采用赋值，所以可以使用分布式锁。

## list

* list 列表可以存储有序，可重复的元素。
* list 列表是一个双向列表，所以获取头部或尾部附近的记录是很快 。

### 常用命令

#### lpush key v1 v2 ……   

从左侧添加多个值到列表

```shell
lpush list_key value1 value2 value3
```

这条命令和下面三条命令的效果是一样的

```shell
lpush list_key value1
lpush list_key value2
lpush list_key value3
```

#### 图解 lpush 命令

![](https://gitee.com/GWei11/picture/raw/master/20210502083250.png)

【说明】：lpush 命令和 接下来要讲的 lpop (从左侧弹出数据) 命令结合的话就相当于是一个栈，因为都是在左侧操作，所以就是 **先进后出**

#### rpush key v1 v2 ……

从右侧加入数据到 list 中

```
rpush list_key v1 v2 v3
```

#### 图解 rpush

![](https://gitee.com/GWei11/picture/raw/master/20210502083212.png)

#### lpop key   

从左侧弹出一个数据

```shell
lpop list_key # 输出的结果 value3
```

从上面的图解可以知道 lpop 是从左侧弹出一个数据，所以弹出的结果就是 value3

#### rpop key

从右侧弹出一个数据

```shell
rpop list_key # 从 rpush 图解哪里可以看出来，从右侧弹出的话结果是 v3
```

#### lpushx key value

lpushx 和 lpush 的作用都是从列表的左侧添加元素，但是不同点在于如果 key 不存在的话，lpush 是会创建一个 list ，而 lpushx 是不会执行成功。

```shell
lpushx list_key lpush_valuex  # 在列表的最左侧添加 lpushx_value
```

#### rpushx key value

rpushx 和 rpush 的作用都是从列表的左侧添加元素，但是不同点在于如果 key 不存在的话，rpush 是会创建一个 list ，而 rpushx 是不会执行成功。

```shell
rpushx list_key rpush_valuex  # 在列表的最右侧添加 rpushx_value
```

#### llen key

获取列表中元素的个数

```shell
llen list_key
```

#### lindex key index

从左侧获取给定下标的元素, 如果后面的 下标的值 为 -1 表示列表里面的最后一个值

```
lindex list_key 0  # 返回值是 通过 lpushx 命令添加的值 lpushx_value
```

#### lrange key startIndex endIndex

返回列表中指定区间的元素，区间是通过 startIndex 和 endIndex 来表示的。

```shell
lrange list_key  0 2 # 返回 lpushx_value value2 value1， 这三个是下标为0， 1,2 的值
```

#### lrem key count value

* 删除列表中与 value 相等的元素
* 当 count > 0 时， lrem 会从列表作品名开始删除，当 count < 0时，lrem 会从列表右边开始删除，当 count= 0时，lrem 会删除所有与 value 相等的元素。

#### lset key index value

将列表 index 位置的元素设置成 value 的值。

#### ltrim key startIndex endIndex

对列表进行修剪，只保留 start 到 end 区间的值

#### rpoplpush key1 key2

从 key1 列表右侧弹出并插入到 key2 列表左侧

#### linsert key BEFORE/AFTER pivot value

将 value 插入到列表，并且位置值 pivot 之前或之后，示例如下

```shell
lrange list_key 0 -1
 1)  "lpushx_value"
 2)  "value2"
 3)  "value1"
 4)  "v1"
 5)  "v2"
 6)  "rpushx_value"
 
linsert list_key BEFORE value2 new_value
"7"

lrange list_key 0 -1
 1)  "lpushx_value"
 2)  "new_value"
 3)  "value2"
 4)  "value1"
 5)  "v1"
 6)  "v2"
 7)  "rpushx_value"
```

### 使用场景

列表有序可以作为栈和队列使用。

* lpush 和 lpop 命令组合其实就是一个栈的效果，因为是先进后出。
* lpush 和 rpop 命令组合其实就是一个队列，是先进先出。

## set

set 中元素是无序、并且元素是惟一的。

### 常用命令

#### sadd key value1 value2 ……

为集合中添加元素

```shell
sadd set_key  value1 value2 value3  # 这里的 set_key 就是自己指定的 key，后面的就是value
```

#### srem key value1 value2 ……

删除集合中指定的值

```shell
srem set_key value2 value4  # 只会删除 value2, 因为集合中没有 value4 这个值
```

#### smembers key

获取集合中所有元素的值

```shell
smembers set_key
```

#### spop key

随机返回集合中一个元素，并且将该元素删除

```shell
spop set_key
```

#### srandmember key

随机返回集合中一个元素，不会删除该元素

```shell
srandmember set_key
```

#### scard key

获取集合中元素的数量

```shell
scard set_key
```

#### sismember key value

判断元素是否在集合中

```shell
sismember set_key value1 # 判断 value1 是否在 集合中，如果在返回 1，不在返回 0
```

#### sinter key1 key2 ……

求多个集合的 **交集**

#### sdiff key1 key2 …… 

求多个集合的 **差集**

#### sunion key1 key2 ……

求多个集合的**并集**

## sortedset

* 存储的元素本身是无序且不重复的
* 每一个元素会关联一个分数 score， 可以按照分数来排序，分数可以重复

### 常用命令

#### zadd key score1 value1 score2 value2 ……

为有序集合添加元素

```shell
zadd zset_key 1 value1 2 value2  1.4 value3 # 添加了三个元素到集合，每一个元素前面的数字就是对应的分数
```

#### zrem key value1 value2 ……

删除有序集合中指定的成员

```shell
zrem zset_key value2  # 返回的数字是删除的数量
```

#### zcard key

获取有序集合中的元素数量

```shell
zcard zset_key
```

#### zcount key min max

返回集合中 score 值在 [min max] 区间的元素数量

```shell
zcount zset_key 0 1
```

#### zincrby key increment value

给集合中指定的元素添加分数

```shell
zincrby zset_key 0.3 value1  # 给集合中 value1 这个值的分数增加 0.3 的分数
```

#### zscore key vlaue

获取集合中 value 的分数

```shell
zscore zset_key value1
```

#### zrank key value

获取集合中 value 的排名，按照分数从小到大的顺序

```shell
zrank zset_key value1 # 顺序是从0开始的，也就是分数最小的排名是0
```

#### zrevrank key value

获取集合中 value 的排名，按照分数从大到小的顺序

```
zrevrank zset_key value1 # 顺序是从0开始的，也就是分数最大的排名是0
```

#### zrange key startIndex endIndex

获取指定区间的成员，按照分数递增来排序

```shell
zrange zset_key 0 -1 # 0 -1 就是查询集合中所有的元素了。
```

#### zrevrange key startIndex endIndex

获取集合中指定区间的成员，按照分数递减排序

```shell
zrevrange zset_key  0 -1 # 这个结果就和 zrange 的结果是相反的顺序
```

### 使用场景

因为可以按照分值来排序，所以适用于各种排行榜，比如，点击量的排行榜，销量的排行榜等。

## hash

hash 是一个string 类型的 field 和 value 的映射表。

### 常用命令

#### hset key field vlaue

给字段设置值，如果已经有了 该 field，就是更新，没有就是新增

```shell
hset h_key username hehe # key 是 h_key, field 是 username ，对应的值是 hehe
```

#### hmset key field1 value1 field2 value2 ……

批量设置

#### hsetnx key field value

如果 fileld 不存在的时候，则进行赋值，如果存在就不做任何操作。

#### hexists key field

查看某个 field 是否存在

#### hget key field

获取一个字段值

#### hmget key field1 field2 ……

获取多个字段的值

#### hgetall key

获取所有字段的值

```shell
hset h_key username hehe
hset h_key age 11
hgetall h_key
```

执行完上面三个命令之后，结果如下：

```shell
 1)  "username"
 2)  "hehe"
 3)  "age"
 4)  "11"
```

#### hdel key field1 field2 ……

删除指定字段

#### hlen key

获取字段数量



## stream （redis5 中新增的一种类型）





上面是五种常见的数据类型，还有两种是不常用的：

* bitmap 位图类型
* geo 地址位置类型

# redis 中的 key 设计

* 使用 `:` 进行分割
* 使用表名 作为 key 的前缀
* 第二部分使用主键 id 
* 第三部分一般使用 列名

【示例】：`user:1001:username`





