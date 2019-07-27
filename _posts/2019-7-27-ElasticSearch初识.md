---
layout:     post
title:      ElasticSearch初识
subtitle:   
date:       2019-7-27
author:     BY KiloMeter
header-img: img/2019-7-27-ElasticSearch初识/64.png
catalog: true
tags:
    - ElasticSearch
---

### 索引增删改查

#### 创建索引

格式：PUT /索引名/Type名/id

PUT /ecommerce/produce/1
{
  "name" : "gaolujie yagao",
  "desc" : "gaoxiao meibai",
  "price" : 30,
  "producer" : "gaolujie producer",
  "tags" : ["meibai","fangzhu"]
}

#### 删除索引

DELETE /ecommerce/produce/1

#### 修改索引

修改索引有两种方式，一种是覆盖，一种是修改

首先，覆盖方式和创建索引的方式一样，json串的内容修改成你想要修改的内容然后选择创建即可覆盖掉旧索引。这种方式的缺点就是，如果只想要修改其中某个字段，需要把原来的字段给复制过来，修改掉要修改的字段后再进行创建，比较麻烦，另一种方式就比较简单，能够只修改某个字段而不会进行覆盖

POST /ecommerce/produce/1/_update
{
  "doc":{
    "add":"add"
  }
}

提交方式为POST，在doc中写入要修改的字段名和值即可。

#### 查找

##### query Search

GET  /索引名/Type名/_search

这是某个查询结果，分析下各个字段的含义

```json
{
  //took 表示查询所花费的时间
  "took" : 0,
  //time_out 是否超时
  "timed_out" : false,
  "_shards" : {
    //shards是es中的分区，这里总共有1个分区
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    // total 命中情况
    "total" : {
      // 命中的数量
      "value" : 3,
      "relation" : "eq"
    },
    //对于一个搜索结果相关度的匹配分数，值越大则相关度越大
    "max_score" : 1.0,
     //下面是命中的各个结果
    "hits" : [
      {
        "_index" : "ecommerce",
        "_type" : "produce",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "name" : "jiajieshi yagao",
          "desc" : "youxiao fangzhu",
          "price" : 25,
          "producer" : "jiajieshi producer",
          "tags" : [
            "fangzhu"
          ]
        }
      },
      {
        "_index" : "ecommerce",
        "_type" : "produce",
        "_id" : "3",
        "_score" : 1.0,
        "_source" : {
          "name" : "test",
          "desc" : "gaoxiao meibai",
          "price" : 30,
          "producer" : "gaolujie producer",
          "tags" : [
            "meibai",
            "fangzhu"
          ]
        }
      },
      {
        "_index" : "ecommerce",
        "_type" : "produce",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "name" : "test",
          "desc" : "test",
          "add" : "add"
        }
      }
    ]
  }
}
```

GET /ecommerce/produce/_search?q=name:yagao&sort=price:desc

这条查询是查询字段为name中有yagao的document并且按照price进行降序，没有:desc的话默认是升序

##### query DSL

这种查询方式，以json格式构造查询语法，比较方便

这条语句是查询所有document

```json
GET /ecommerce/produce/_search
{
  "query": {
    "match_all": {}
  }
}
```

这条查询和前面GET /ecommerce/produce/_search?q=name:yagao&sort=price:desc的结果是一样的

```json
GET /ecommerce/produce/_search
{
  "query": {
    "match": {
      "name":"yagao"
    }
  },
  "sort": [
    {
      "price": {
        "order": "desc"
      }
    }
  ]
}
```

from和size指定要查询条数的起始位置和条数

```json
GET /ecommerce/produce/_search
{
  "query": {"match_all": {}},
  "from":1,
  "size":2
}
```

source中指定要查询的某些字段

```json
GET /ecommerce/produce/_search
{
  "query": {"match_all": {}},
  "_source": ["name","desc"]
}
```

##### query filter

```json
GET /ecommerce/produce/_search
{
  "query": {
    //bool中可以封装多个查询条件
    "bool": {
      //必须匹配
      "must": [
        {"match": {
          "name": "yagao"
        }}
      ],
      //过滤
      "filter": {
        //范围
        "range": {
          //price字段要大于25
          "price": {
            "gt": 25
          }
        }
      }
    }
  }
}

```

##### full-text search (全文检索)

```json
GET /ecommerce/produce/_search
{
  "query": {
    "match": {
      "producer": "yagao producer"
    }
  }
}
```

##### phrase search(短语搜索)

跟全文检索相对，全文检索中，会把搜索串拆解开来，去倒排索引中一一匹配。

而短语搜索需要搜索串完全匹配上才能被检索出来

```json
GET /ecommerce/produce/_search
{
  "query": {
    "match_phrase": {
      "producer": "yagao producer"
    }
  }
}
```

##### highlight search(高亮搜索结果)

```json
GET /ecommerce/produce/_search
{
  "query": {
    "match": {
      "producer": "yagao producer"
    }
  },
  "highlight": {
    "fields": {
      "producer":{}
    }
  }
}
```

### 聚合操作

#### 计算每个tag下的商品数量

```json
GET /ecommerce/produce/_search
{
  //这里是把查询的结果个数设置为0，这样的话就只会显示聚合结果
  "size":0,
  //"aggs"指定聚合结果
  "aggs": {
    //这里为聚合结果起个名字
    "group_by_tags": {
      //term是一种操作，含义是按照下面field指定的字段进行分组，并且输出分组的数量
      "terms": {
        //field字段指定了查询结果要按照哪个字段进行聚合
        "field":"tags"
      }
    }
  }
}
```

如果直接运行的话会报错，需要先修改下参数，把fielddata设置为true

```json
PUT /ecommerce/_mapping
{
  "properties": {
    "tags": {
      "type": "text",
      "fielddata": true
    }
  }
}
```

#### 指定查询要求的聚合操作

和上面的聚合类似，只需要加入匹配字段即可

```json
GET /ecommerce/produce/_search
{
  //查询字段
  "query": {
    "match": {
      "name": "jiajieshi"
    }
  }, 
  "aggs": {
    "group_by_tags": {
      "terms": {
        "field":"tags"
      }
    }
  }
}
```

#### 分组计算平均值

```json
GET /ecommerce/produce/_search
{
  
  "aggs": {
    "group_by_tags": {
      "terms": {
        "field":"tags"
      },
      "aggs":{
        "avg_price":{
          "avg":{
            "field":"price"
          }
        }
      }
    }
  }
}
```

#### 分组计算平均值并排序

```json
GET /ecommerce/produce/_search
{
  
  "aggs": {
    "group_by_tags": {
      "terms": {
        "field":"tags",
        //指定排序
        "order":{
          "avg_price":"desc"
        }
      },
      "aggs":{
        "avg_price":{
          "avg":{
            "field":"price"
          }
        }
      }
    }
  }
}

```

