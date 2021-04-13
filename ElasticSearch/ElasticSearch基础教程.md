# Elasticsearch 的基本概念

**Elasticsearch 基本介绍** https://zhuanlan.zhihu.com/p/62892586 

Elasticsearch 是一个全文搜索引擎. 平时我们将数据存在数据库中也是可以搜索一样, 比如 Mysql 中有 Database, Table, Column 等概念, 对于 Elasticsearch 其实也是一样的, 学习任何一个东西需要先了解它的定义以及它包含的内容的定义. 下面就来说一下 ELasticsearch 的中的几个概念.

* 索引 ( index )

* 映射 ( mapping )
* 文档 ( doc )

## 索引

* 索引类似于关系型数据库中的 database 的概念.
* 类似的数据放在一个索引里面, 不同类型的数据放在不同类型的索引里面. 
* 可以理解为索引就是 Elasticsearch 中最外层的容器.

## 映射

* 映射可以理解为关系型数据库中的表结构. 既然是结构就说明是一种约束.
* 从上面的索引我们可以知道同类数据放在相同索引里面, 那你怎么知道两个数据来了他们是同一类呢? 这就说明是需要一个约束的东西, 映射就是表述这个约束关系的.

## 文档

* 文档可以对应为关系型数据库中的一行数据.
* 无论是索引还是映射都是说 Elasticsearch 这个组件自身的一些结构, 而文档则是用来存储外部数据的.



# Elasticsearch 集成 IK 分词器

## 为什么要安装分词器

问这个问题时就要我们明白 Elasticsearch 是如何做到全文搜索的, 从上面第一行的链接中进入的文章里举了一个例子, 让你说出带有 `前`  这个字的诗句, 这你一下子很难想的起来. 但是如果给出了这样的一个映射

```xml
前 ------> 床前明月光
```

那你根据 `前` 这个 key  立马就可以找到对应的 `value`窗前明月光, 这是一种倒排索引的运用. 

对于倒排索引最极致的做法就是给你一篇文章的时候, 比如给你一句话: `我要好好学习, 天天向上.` 现在你可以将所有不同的字都拆分出来与这句话进行对应.

```xml
我 -------------> 我要好好学习, 天天向上
要 -------------> 我要好好学习, 天天向上
好 -------------> 我要好好学习, 天天向上
学 -------------> 我要好好学习, 天天向上
习 -------------> 我要好好学习, 天天向上
天 -------------> 我要好好学习, 天天向上
向 -------------> 我要好好学习, 天天向上
上 -------------> 我要好好学习, 天天向上
```

这么一对应, 是不是可以根据任何一个字都能找到对应的这句话呢? 当然实际上肯定不是一个字一个字的对应了

* 有些内容本来就应该是连起来的, 比如上面那句话中的 `学习`  两个字.
* 还有一些字是不需要给索引的, 比如 `的`, `了` 等辅助词。

既然不能那么极端，给你一篇文章就按照一个字一个字的来倒排索引，那就肯定要有其他方式来进行拆分了，分词器就是用来做这个的。下面就来安装一下分词器。

## 下载插件并安装

**【注意】**：因为 Elasticsearch 不能使用 root 用户启动，所以这里也是使用启动 Elasticsearch 用户的那个用户来操作。

* 插件地址 https://github.com/medcl/elasticsearch-analysis-ik/ 

![](https://gitee.com/GWei11/picture/raw/master/20210411091917.png)

**【注意】**打开 github 地址后，不是直接从 code 那里下载，而是往下看 install 那里，有两种安装方式，如果使用第一种也就是先下载文件下来本地安装的话，需要从这个链接进去下载文件，如果是直接从 code 那里下载，安装的时候会出现如下错误

```shell
Exception in thread "main" java.nio.file.NoSuchFileException: /usr/elasticsearch/plugins/.installing-2567752034376027906/plugin-descriptor.properties
```

* 解压插件

```shell
cd /usr/elasticsearch/plugins  # 进入elasticsearch安装目录里面的plugins目录
mkdir ik # 在 plugins目录里面创建一个目录
# 然后将下载的ik分词器压缩包放到创建的 ik 目录里面
unzip elasticsearch-analysis-ik-7.3.0.zip # 解压
```

* 安装插件（当前目录在 /usr/elasticsearch/plugins/ik 中）

```shell
/usr/elasticsearch/bin/elasticsearch-plugin install elasticsearch-analysis-ik-7.3.0.zip
```

如果，不是在这个目录里面，可以使用下面的命令来进行安装

```shell
/usr/elasticsearch/bin/elasticsearch-plugin install file:///usr/elasticsearch/plugins/elasticsearch-analysis-ik-7.3.0.zip
```

* 安装完成之后，重新启动 Elasticsearch 和 Kibana 即可。

## 分词器的两种分词模式

IK 分词器有两种分词模式

* ik_max_word： 这个分词器会将文本做最细粒度的拆分

先安装Elasticsearch [在Centos7中安装 Elasticsearch](https://mp.weixin.qq.com/s?__biz=Mzg3NDYwNjA4Ng==&mid=2247483714&idx=1&sn=9bd22e539ab4e1ee1dfb317099e7ccd0&chksm=cecf7a2bf9b8f33dffa62fe09361adb7e7d5aa1b736e4215a0a18330c528c4f0997d2a0edff6&scene=21&token=792123018&lang=zh_CN#wechat_redirect) ，然后启动 Elasticsearch 和 Kibana （测试前提是已经安装了上面的分词器），我们来测试一下。

```http
POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "我正在学习的是ELasticsearch这个课程"
}
```

结果

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "正在",
      "start_offset" : 1,
      "end_offset" : 3,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "在学",
      "start_offset" : 2,
      "end_offset" : 4,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "学习",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 3
    },
    {
      "token" : "的",
      "start_offset" : 5,
      "end_offset" : 6,
      "type" : "CN_CHAR",
      "position" : 4
    },
    {
      "token" : "是",
      "start_offset" : 6,
      "end_offset" : 7,
      "type" : "CN_CHAR",
      "position" : 5
    },
    {
      "token" : "elasticsearch",
      "start_offset" : 7,
      "end_offset" : 20,
      "type" : "ENGLISH",
      "position" : 6
    },
    {
      "token" : "这个",
      "start_offset" : 20,
      "end_offset" : 22,
      "type" : "CN_WORD",
      "position" : 7
    },
    {
      "token" : "课程",
      "start_offset" : 22,
      "end_offset" : 24,
      "type" : "CN_WORD",
      "position" : 8
    }
  ]
}
```

* il_smart：会做最粗粒度的拆分

依然是使用上面的例子

```http
POST _analyze
{
  "analyzer": "ik_smart",
  "text": "我正在学习的是ELasticsearch这个课程"
}
```

结果

```json
{
  "tokens" : [
    {
      "token" : "我",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "CN_CHAR",
      "position" : 0
    },
    {
      "token" : "正在",
      "start_offset" : 1,
      "end_offset" : 3,
      "type" : "CN_WORD",
      "position" : 1
    },
    {
      "token" : "学习",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "CN_WORD",
      "position" : 2
    },
    {
      "token" : "的",
      "start_offset" : 5,
      "end_offset" : 6,
      "type" : "CN_CHAR",
      "position" : 3
    },
    {
      "token" : "是",
      "start_offset" : 6,
      "end_offset" : 7,
      "type" : "CN_CHAR",
      "position" : 4
    },
    {
      "token" : "elasticsearch",
      "start_offset" : 7,
      "end_offset" : 20,
      "type" : "ENGLISH",
      "position" : 5
    },
    {
      "token" : "这个",
      "start_offset" : 20,
      "end_offset" : 22,
      "type" : "CN_WORD",
      "position" : 6
    },
    {
      "token" : "课程",
      "start_offset" : 22,
      "end_offset" : 24,
      "type" : "CN_WORD",
      "position" : 7
    }
  ]
}
```



# Elasticsearch 的基本 API

Elasticsearch 提供了基于 http 请求的 Rest 风格的 API, 也提供了各种语言的客户端 API.

下面都是使用 Rest 风格的 API 进行说明

【注意】：所有操作的方法比如 PUT，DELETE，POST等都要大写（使用 kibana 的时候）。

## 操作索引的API

### 新增

语法：

```json
PUT /索引名称
{
    "settings": {
    	"属性名": "属性值"
    }
}
```

settings 就是用来索引的元数据。可以定义索引的各种属性，比如分片数， 副本等。不设置的话就使用默认值。

下面新增一个 index_one 的索引。

示例：

```json
PUT /index_one
```

创建的结果

![](https://gitee.com/GWei11/picture/raw/master/20210413060424.png)

我们可以试一下使用 POST 来创建索引， 结果如下

![](https://gitee.com/GWei11/picture/raw/master/20210413060519.png)

右侧看到已经报错了，说的是不允许使用 POST 方法，对索引的操作被允许的方法有 PUT，GET，HEAD，DELETE。

这是为什么呢？这就涉及到了**幂等性** 这个概念了。幂等性多次相同的请求结果都是一样的 ，具有幂等性的几个方法是 PUT，DELETE，HEAD，GET 而 POST 方法是不具有幂等性的。



### 判断索引是否存在

语法：

```json
HEAD /索引名称
```

示例：

```json
HEAD /index_one
# 下面是返回的结果
200 - OK

# 第二条测试语句
HEAD /index_two
# 下面是返回结果
404 - Not Found
```

### 删除

语法：

```json
# 多个索引之间不能含有空格，否则只会删除第一个索引
DELETE /索引名称1,索引名称2……
```

### 查询

* 查询单个索引语法：

```json
GET /索引名称
```

示例：

```json
GET /index_one
```

结果：

```json
{
  "index_one" : {
    "aliases" : { },
    "mappings" : { },
    "settings" : {
      "index" : {
        "creation_date" : "1618264862751",
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        "uuid" : "QGZbOdz5RoyS8zjZagK8qQ",
        "version" : {
          "created" : "7030099"
        },
        "provided_name" : "index_one"
      }
    }
  }
}
```



* 批量查询索引

```json
# 第一种方式， 使用这种方式多个索引名称之间使用英文逗号并且不能有空格
GET /索引名称1，索引名称2…… 
# 第二种方式
GET _all
# 第三种方式
GET /_cat/indices?v
```

#### 打开索引

语法：

```json
POST /索引名称/_open
```

#### 关闭索引

语法：

```json
POST /索引名称/_close
```

## 操作映射的API

### 创建映射字段

语法：

```json
PUT /索引名称/_mapping
{
    "properties": {
        "字段名": {
            "type": "类型",
            "index": true,
            "store": true,
            "analyzer": "分词器"
        }
    }
}
```

* `字段名` 类似于我们在 Mysql 中创建的字段名一样，可以任意定义、
* type：用来描述该字段的类型， 支持的类型有很多
  * String类型
    * text： 支持分词，不可以参与聚合
    * keyword：不可以分词，数据完整保存，可以参与聚合
  * Numerical：数值类型
    * 基本数据类型：long、interger、short、byte、double、float、half_float
    * 浮点数的高精度类型：scaled_float
  * Date：日期类型，可以对日期转换为字符串后存储，也可以存储为毫秒值（建议）
  * Array：数组类型
  * Object： 对象
* index：是否索引，默认为true
  * 为 true 可以被索引，也就是可以被用来搜索
  * 为 false 不被索引，不可以被用来搜索
* store：是否需要**独立**存储，默认true
  * 原始文本会存在一个地方，store 属性开启后该字段的内容会在另外一个地方存储一份数据，获取独立存储的内容比在原来的地方从快，但是会占用更多的空间。
* analyzer：指定分词器，比如可以指定 ik 分词器的 ik_smart，ik_max_word等。

示例：

```json
PUT /index_one
PUT /index_one/_mapping/
{
  "properties": {
    "name": {
      "type": "text",
      "analyzer": "ik_max_word"
    },
    "job": {
      "type": "text",
      "analyzer": "ik_max_word"
    },
    "logo": {
      "type": "keyword",
      "index": "false"
    },
    "payment": {
      "type": "float"
    }
  }
}
```

上面的例子中设置了四个字段

* name
* job
* logo
* payment

### 查看索引映射关系

#### 单个索引映射

语法：

```json
GET /索引名称/_mapping
```

示例：

```json
GET /index_one/_mapping
```

结果：（前提条件是创建了映射）

```json
{
  "index_one" : {
    "mappings" : {
      "properties" : {
        "job" : {
          "type" : "text",
          "analyzer" : "ik_max_word"
        },
        "logo" : {
          "type" : "keyword",
          "index" : false
        },
        "name" : {
          "type" : "text",
          "analyzer" : "ik_max_word"
        },
        "payment" : {
          "type" : "float"
        }
      }
    }
  }
}
```

#### 查看所有索引映射关系

语法：

```json
# 方式1
GET _mapping
# 方式2
GET _all/_mapping
```

#### 修改索引映射

语法：

```json
PUT /索引库名/_mapping
{
  "properties": {
    "字段名": {
      "type": "类型",
      "index": true,
      "store": true,
      "analyzer": "分词器"
    }
  }
}
```

【注意】：修改映射字段可以，如果是做其他修改只能先删除索引，然后重新创建映射

## 同时操作索引和映射

语法：

```json
PUT /索引名称
{
  "settings": {
    "索引库属性名": "索引库属性值"
  },
  "mappings": {
    "properties": {
      "字段名": {
        "映射属性名": "映射属性值"
      }
    }
  }
}
```

示例：

```json
PUT /index_two
{
  "mappings": {
    "properties": {
      "name": {
        "type": "text", 
        "analyzer": "ik_max_word"
      }
    }
  }
}
```

上面示例中没有给 settings 设置值，说明就是使用默认值。

## 操作文档的API

### 新增文档（手动指定id）

语法：

```json
POST /索引名称/_doc/{id}
{
    "field": "value"
}
```

示例：

```json
POST /index_two/_doc/1
{
  "name": "张三"
}
```

* 里面的 “name” 就是在创建映射的时候的 **字段名**，就好像我们在 mysql 中创建表结构的时候只有一个 name 字段，那么新增数据的时候就只能用这个 name 字段来存值了。（当然在 Elasticsearch 中宽容一点，可以通过设置来改变这个形式，即使在创建映射的时候没有该字段，也可以让文档数据添加成功）

### 新增文档（自动生成id）

语法：

```json
POST /索引名称/_doc/
{
    "field": "value"
}
```

示例：

```json
POST /index_two/_doc/
{
  "name": "李四"
}
```

结果：

![](https://gitee.com/GWei11/picture/raw/master/20210413065906.png)



### 查看单个文档

语法：

```json
GET /索引名称/_doc/{id}
```

示例：

```json
GET /index_two/_doc/1
```

结果：

```json
{
  "_index" : "index_two",
  "_type" : "_doc",
  "_id" : "1",
  "_version" : 3,
  "_seq_no" : 2,
  "_primary_term" : 1,
  "found" : true,
  "_source" : {
    "name" : "张三"
  }
}
```

结果元数据解析：

_index： 文档所属的 index

_type： 文档所属的type， Elasticsearch7以后默认都是 _doc

_id：文档的唯一标识

_version: 文档的版本号

_seq_no: 严格递增的顺序号，每个文档一个

_primary_term: 任何类型的写操作，包括index，create，update和delete都会生成一个 _seq_no

found: true/false， 表示文档是否找到了

_source: 存储原始文档



### 查看所有文档

语法：

```json
POST /索引名称/_search
{
  "query": {
    "match_all": {}
  }
}
```



### 更新文档（全部更新）

将新增文档的 POST 请求修改为 PUT 请求就是更新了，修改文档需要带上 id

语法：

```json
PUT /索引名称/_doc/{id}
{
  "name": "王五"
}
```

示例：

```json
# 新增文档，当然如果文档已经存在，这个方法也是更新文档
POST /index_two/_doc/1
{
  "name": "李四"
}

# 更新文档
PUT /index_two/_doc/1
{
  "name": "王五"
}
```

### 更新文档（局部更新）

语法：

```json
POST /索引名称/_update/{id}
{
  "doc": {
    "field": "value"
  }
}
```

示例：

```json
POST /index_two/_update/1
{
  "doc": {
    "name": "hello"
  }
}
```

### 删除文档

* 根据 id 来删除

语法：

```json
DELETE /索引名称/_doc/{id}
```

* 根据查询条件进行删除

语法：

```json
POST /索引名称/_delete_by_query
{
  "query": {
    "match": {
      "字段名": "搜索关键字"
    }
  }
}
```

示例：

```json
POST /index_two/_delete_by_query
{
  "query": {
    "match": {
      "name": "hello"
    }
  }
}
```

上面的示例中是根据 name 字段的值为 hello过滤并进行删除