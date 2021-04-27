# 什么是聚合分析

* Elasticsearch 中的聚合分析其实和 Mysql 中的 Group by 是很类似的（可以使用这种方式便于自己理解），是 es 除搜索功能外提供的针对 es 数据做统计分析功能
* es 中的聚合分析从官网也可以看到主要有三种形式
  * Bucket
    * 分桶类型，类似 sql 语句中的 group by 语法
  * Metric
    * 指标分析类型，比如计算最大值，最小值，平均值等
  * Pipeline
    * 管道分析类型，基于上一级的聚合分析结果进行再分析

# 聚合分析API

![](https://gitee.com/GWei11/picture/raw/master/20210428052602.png)

示例：比如现在想找到某一个索引中同名的都有多少人

构造数据

```java
POST /index-name/_doc/1
{
    "name": "王五",
    "age": 18
}

POST /index-name/_doc/2
{
    "name": "王五",
    "age": 19
}

POST /index-name/_doc/3
{
    "name": "赵六",
    "age": 18
}
```

执行查询

```java
GET /index-name/_search
{
  "aggs": {
    "aggs_name": {
      "terms": {
        "field": "name.keyword"
      }
    }
  }
}
```

执行结果

![](https://gitee.com/GWei11/picture/raw/master/20210428053248.png)

## Metric 聚合分析

主要有以下两类

### 单值分析

* min、max、avg、sum
* cardinality
  * 表示不同数值的个数，类似 sql 语句中的 distinct count 概念

#### min

>  返回**数值类**字段最小值

示例：查询的结果是 index-name 这个索引中所有数据的 age 字段的最小值

```java
GET index-name/_search
{
  "aggs": {
    "min_age": {
      "min": {
        "field": "age"
      }
    }
  }
}
```

#### sum

> 返回**数值类**字段的总和

示例：查询的结果是 index-name 这个索引中所有数据的 age 字段的和

```java
GET index-name/_search
{
  "aggs": {
    "sum_age": {
      "sum": {
        "field": "age"
      }
    }
  }
}
```

#### 一次性返回多个聚合结果

示例：就是将多个单值分析结果总和起来，比如将上面的 min 和 sum 综合起来。

```java
GET index-name/_search
{
  "aggs": {
    "sum_age": {
      "sum": {
        "field": "age"
      }
    },
    "min_age":{
      "min": {
        "field": "age"
      }
    }
  }
}
```

输出的结果中包含如下内容

```java
"aggregations" : {
    "sum_age" : {
      "value" : 55.0
    },
    "min_age" : {
      "value" : 18.0
    }
  }
```

#### cardinality

> 其作用就是去重之后再求 count ，类似于 sql 里面的 distinct count

构造测试数据

```java
POST /index-name/_doc/1
{
    "name": "王五",
    "age": 18
}

POST /index-name/_doc/2
{
    "name": "王五",
    "age": 19
}

POST /index-name/_doc/3
{
    "name": "赵六",
    "age": 18
}
```

执行查询

```java
GET index-name/_search
{
  "aggs": {
    "count_of_age": {
      "cardinality": {
        "field": "age"
      }
    }
  }
}
```

返回结果：既然都是单值分析，所以最终的结果都是返回一个值，这里返回2 就是因为 age 字段有 18 和 19 两个类型的值，18本来有两个，但是去重了。

```java
"aggregations" : {
    "count_of_age" : {
      "value" : 2
    }
  }
```



### 多值分析，输出多个分析结果

* stats、extended stats
* percentile，percentile rank
* top hits

#### stats

> 返回一系列数值类型的统计值，包含 min、max、avg、sum 和 count

示例：

```java
GET /index-name/_search
{
  "aggs": {
    "stats_age": {
      "stats": {
        "field": "age"
      }
    }
  }
}
```

返回结果：

```java
"aggregations" : {
    "stats_age" : {
      "count" : 3,
      "min" : 18.0,
      "max" : 19.0,
      "avg" : 18.333333333333332,
      "sum" : 55.0
    }
  }
```

#### extended stats

> 对 stats 的扩展，包含了更多的统计数据，比如 方差，标准差等

示例：

```java
GET /index-name/_search
{
  "aggs": {
    "extended_stats_age": {
      "extended_stats": {
        "field": "age"
      }
    }
  }
}
```

返回结果：

```java
"aggregations" : {
    "extended_stats_age" : {
      "count" : 3,
      "min" : 18.0,
      "max" : 19.0,
      "avg" : 18.333333333333332,
      "sum" : 55.0,
      "sum_of_squares" : 1009.0,
      "variance" : 0.22222222222220958,
      "variance_population" : 0.22222222222220958,
      "variance_sampling" : 0.3333333333333144,
      "std_deviation" : 0.4714045207910183,
      "std_deviation_population" : 0.4714045207910183,
      "std_deviation_sampling" : 0.5773502691896093,
      "std_deviation_bounds" : {
        "upper" : 19.27614237491537,
        "lower" : 17.390524291751294,
        "upper_population" : 19.27614237491537,
        "lower_population" : 17.390524291751294,
        "upper_sampling" : 19.488033871712553,
        "lower_sampling" : 17.17863279495411
      }
    }
  }
```

#### top hits

> 作用是取每一个分好的组里面的顶部的数据

示例：

```java
GET /index-name/_search
{
  "aggs": {
    "ages": { # 这里的ages 是第一层的聚合分析的名字
      "terms": { # 这里的terms 的作用是按照 age 字段分组
        "field": "age",
        "size": 10
      },
      "aggs": { # 注意这个聚合分析是在第一个聚合分析 ages 的里面，所以意思是当 age 字段分好组之后再来做这个分组里面的事情
        "top_age": { # 这是内层聚合分析的名字
          "top_hits": { # 取每一个组里面头部的值
            "size": 2,  # 这里为2就表示每一个组里面取前面两个
            "sort": [{ # 这里表示每一个组按照 name 字段倒序排序
              "name": {
                "order": "desc"
              }
            }]
          }
        }
      }
    }
  }
}
```

![](https://gitee.com/GWei11/picture/raw/master/20210428061703.png)

## Bucket 聚合分析

bucket 就是按照一定的规则将文档分配到不同的桶中，达到分类的目的。常见的分桶策略有以下几种

* Terms 
* range
* date range
* histogram
* date histogram



### terms

> 直接按照单词来进行分桶，如果是 text 类型，则按照分词后的结果来进行分桶。

示例：

```java
GET /index-name/_search
{
  "aggs": {
    "names": { # 这个是自己取的分桶名称
      "terms": { # 分桶的策略是 terms
        "field": "name.keyword",  # 这里是使用name 字段来进行分桶，同时使用了 keyword ,也就是不进行分词
        "size": 10
      }
    }
  }
}
```

【注意】：如果是针对 text 类型字段做分桶，那么需要设置分桶的字段 的 `fielddata=true` ，否则会报错

### range

> 按照给定的范围来进行分桶

示例：

```java
GET /index-name/_search
{
  "aggs": {
    "range-age": {
      "range": {
        "field": "age",
        "ranges": [
          {
            "to": 18  # age < 18
          },
          {
            "from": 19,  # age >= 19 and age < 20
            "to": 20
          },
          {
            "from": 21  # age >= 21
          }
        ]
      }
    }
  }
}
```



## Pipeline 聚合分析

> 针对聚合分析的结果再次进行聚合分析，而且支持链式调用

Pipeline 的分析结果会输出到原结果中，根据输出位置的不同，可以分为以下两类

* parent 结果内嵌到现有的聚合分析结果中
  * Derivative
  * Moving Average
  * Cumulative Sum
* Sibling 结果与现有聚合分析结果同级
  * Max、Min、Avg、Sum、Bucket
  * Stats、Extended Stats Bucket
  * Percentiles Bucket

添加测试数据

```java
POST /index-name/_doc/
{
    "name": "测试3",
    "age": 21,
    "salary": 14  # 更换 name 和 salary 的值多执行几次
}


POST /index-name/_doc/
{
    "name": "赵六3",
    "age": 18,
    "salary": 23 # 更换 name 和 salary 的值多执行几次
}

POST /index-name/_doc/
{
    "name": "呵呵1",
    "age": 19,
    "salary": 32 # 更换 name 和 salary 的值多执行几次
}
```

查询示例：

```java
GET /index-name/_search
{
  "aggs": {
    "ages": {
      "terms": {  # 第一次分桶是按照年龄来分组
        "field": "age",
        "size": 10
      },
      "aggs": {
        "avg_ages": { # 按照年龄分组后，求每一个组的平均工资
          "avg": {
            "field": "salary"
          }
        }
      }
    },
    "min_salary_by_age": {
      "min_bucket": { # 获取平均工资最小的那个组的平均工资的值
        "buckets_path": "ages>avg_ages"
      }
    }
  }
}
```

* avg_ages 是 ages 的子分组， 所以 avg_ages 聚合分析是对 ages 分组后的结果进行处理
* min_salary_by_age 和 ages 是同级的，min_salary_by_age 就是一个 pipline ，是拿到 ages 的结果后再一次进行处理