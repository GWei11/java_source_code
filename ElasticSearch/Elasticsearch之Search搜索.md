# Search API

Search 的作用是用来查询 Elasticsearch 中的数据，可以理解为数据库中 SQL 查询语言。

## 主体查询形式

### URI Search

可以理解为 GET 请求，这种就是直接将请求参数放入到 url 中。

```java
GET /index-name/_search
GET /index-name,index-name2/_search  # 一次性查询多个 索引
GET /index-*/_search     # 使用通配符来一次性查询多个索引
```

上面这种是不带参数的查询，当带上参数的时候，常用的参数有下面几个

* q：执行查询的语句
* df：当 q 中不指定字段时默认查询的字段，如果不指定， es 会查询所有字段
* sort：排序
* timeout：指定超时时间，默认不超时
* from，size：用于分页

比如下面的案例表示的意思如下：【q =张三】表示要查询的内容值是张三，【df=name】这个和前面的 q=张三 搭配起来使用，表示 name 字段的值等于 张三，【sort=age】表示查询结果按照 age 进行排序，【from=0】表示从 0 号位置开始查找， 【size=2】表示每一页有两条结果，【timeout=1s】表示该查询语句的超时时间为 1s

```http
POST /index-name/_doc/1
{
    "name": "张三",
    "age": 18
}

# 上面是用于创建文档，下面是执行查询

GET /index-name/_search?q=张三&df=name&sort=age&from=0&size=2&timeout=1s
```

#### 使用 profile 查看真实查询内容

```java
GET /index-name/_search?q=张三  # 如果我们想要知道这个语句在 Elasticsearch 中具体是如何去查询的，那么可以使用下面的形式，使用 profile

GET /index-name/_search?q=张三
{
  "profile": "true"
}
```

使用 profile 之后，在展示的结果中会知道这个语句在 Elasticsearch 中具体是如何执行的

![](https://gitee.com/GWei11/picture/raw/master/20210427215909.png)

### Request Body Search

使用这种方式 es 提供完备查询语法 Query DSL（Domain Specific Language），这种方式是将查询语句通过 http request body 发送到 es

比如：

```java
GET /index-name/_search
{
  "query": {
    "term": {
      "name": "张三"
    }
  }
}
```

这种形式是 基于 Query DSL 进行查询的，下面就来讲解 Query DSL。

### Query DSL

这是一种基于  JSON 定义的查询语言，主要包含如下两种类型

* 字段类查询，比如 term， match， range 等，只针对某一个字段进行查询，分为两种

  * 全文匹配
    * 针对 text 类型字段进行全文检索，会对查询语句先进行分词处理，比如 match， match_phrase （有顺序要求的查询）等 query 类型

  * 单词匹配
    * 不会对查询语句做分词处理，直接去匹配字段的倒排索引，比如 term，terms，range 等 query 类型。

* 复合查询

  * 比如 bool 查询等，包含一个或者多个字段类型查询或者复合查询语句，主要包含以下几类
  * constant_score_query
  * bool query，不二查询由一个或多个布尔子句组成，主要包含下面几个
    * filter：只过滤符合条件的文档，不计算相关性得分
    * must：文档必须符合 must 中的所有条件，会影响相关性得分
    * must_not：文档必须符合 must_not 中的所有条件
    * should：文档可以符合 should 中的条件，会影响相关性得分
  * function_score_query
  * boosting_query

#### 全文匹配

```java
POST /index-name/_doc/1
{
    "name": "张三 李四",
    "age": 18
}
# 上面是新增文档数据， 下面执行查询
GET /index-name/_search
{
  "query": {
    "match": {
      "name": "张 三"
    }
  }
}
```

使用上面的查询代码是可以查询出结果的。我们可以先使用下面的代码看一下新增的那个数据的 name 字段的分词结果是什么

```java
GET /index-name/_doc/1/_termvectors?fields=name
```

看到分词结果是将 “张三 李四” 最终分成了 “张”，“三”，“李”，“四”，其实也就不难看出我们查询的使用的是 “张 三”，那么也会先进行分词为 “张”，“三”，所以肯定是可以查询出来的，实际上就算使用 “张 五”等也是可以查询出来的，因为包含了 “张” 字。

#### operator 参数

上面我们说即使使用“张 五”也是可以查询出来的，就是因为虽然分成了 “张” 和 “五”，但是 “张” 是包含在文档中的，所以可以查询出来，那么怎么样做可以让查询的结果既包含 “张” 又包含 “五” 呢？ 答案就是使用 operator 参数配置 or 和 and 来使用。

```java
GET /index-name/_search
{
  "query": {
    "match": {
      "name": {
        "query": "张 五",
        "operator": "and"
      }
    }
  }
}
```

 使用上面的查询语句就查询不到结果。因为上面查询语句的意思是既要包含 “张”, 又要包含 “五”。