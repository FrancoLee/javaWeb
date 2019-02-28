## ElasticSearch

---

`primay shard` :一个`lucene`的实例，创建索引时就已经固定。

`replica  shard`:`primary shard`的副本

### 常用参数

---

`http://localhost:9200/?pretty`,通过网页查看当前节点信息

+ `name`：当前节点名称

+ `cluster_name`:所在集群名称（默认`elasticsearch`）

+ `number`:版本号

+ `lucene_version`:`lucene`版本号

`GET` 表示以`GET`方式向`elasticsearch`服务器发送请求

`GET /_cat/health?v`：查看集群当前基本信息

+ `status`:当前集群状态(`green`,`yellow`,`red`)
  1. `green`:集群状态良好，所有功能正常
  2. `yellow`:部分的`replica shard`没有正确的分发。(`primary shard`是良好的，意味着集群所有功能还是正常的)
  3. `red`:`primary shard`和对应的`replica shard`数据出现错误，但是还能处理部分分片处理搜索请求。
+ `cluster`:集群名称
+ `node.total`:节点个数
+ `shards`:分片数量
+ `active_shards_percent`：活跃的`shards`百分比

`GET /_cat/indices?v` ：查看索引信息

+ `health`:索引当前状态(`green`,`yellow`,`red`)
+ `index`:索引名字
+ `pri`:`primary shard`分片数量
+ `rep`:`replica shard`分片数量
+ `docs.count`:文档数量
+ `docs.deleted`：删除的文档数量
+ `store.size`：数据项所占用大小
+ `pri.store.size`:`primary shard`所占用大小

`PUT /index_name?pretty `:创建名为`index_name`的索引

`DELETE /index_name?pretty`：删除名为`index_name`的索引

`PUT /index/type/id`：在`index`下创建`type`类型的文档，其`_id`值为`id`(再次执行表示覆盖，覆盖必须覆盖文档的所有值)

```elixir
PUT /web/article/1 
{
  "title":"title",
  "centext":"text",
  "src":" "
}
/**
返回结果
{
  "_index" : "web",
  "_type" : "article",
  "_id" : "1",
  "_version" : 1,
  "result" : "created",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 0,
  "_primary_term" : 1
}
**/
```

+ `_shards`:分片信息，`total`表示要写入几个分片，`successful`,成功写入了几个分片。在这里要写入`primary shard`和`replica shard`,而现在只有一个`primary shard`服务器启动了，所以`successful`为`1`；
+ `version`:版本号，用于锁
+ `_seq_no`:`shard`级别严格递增的顺序号，保证后写入的`doc`的`_seq_no`大于先写入的。作用(mark)
+ `_primary_term`:当`primary shard`重新分配时会递增`1`。(mark)

`GET /index/type/id`:获取`index`索引下`type`类型`_id`值为`id`的`doc`

```elixir
GET /web/article/1
/** 
返回结果
{
  "_index" : "web",
  "_type" : "article",
  "_id" : "1",
  "_version" : 6,
  "found" : true,
  "_source" : {
    "title" : "title",
    "centext" : "text",
    "src" : ""
  }
}
**/
```

​	字段意思看英文就能猜到

`POST /index/type/id/_update {"doc":{source}}`:部分修改

```elixir
POST /web/article/1/_update
{
  "doc":{
    "title":"ss"
  }
}
/**
返回结果
{
  "_index" : "web",
  "_type" : "article",
  "_id" : "1",
  "_version" : 7,
  "result" : "updated",
  "_shards" : {
    "total" : 2,
    "successful" : 1,
    "failed" : 0
  },
  "_seq_no" : 6,
  "_primary_term" : 1
}
**/
```

`DELETE /index/type/id`:删除文档

----

搜索(6种)：

+ `query string search`:
+ `query DSL`
+ `query filter`
+ `full-text search`
+ `phrase search`
+ `highlight search`

----

`query string search`:

`GET /index/type/_search`:搜索`index`下`type`的所有数据

```elixir
GET /web/article/_search
/**
{
  "took" : 7,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "web",
        "_type" : "article",
        "_id" : "sad",
        "_score" : 1.0,
        "_source" : {
          "title" : "title2",
          "centext" : "text2",
          "src" : ""
        }
      },
      {
        "_index" : "web",
        "_type" : "article",
        "_id" : "2",
        "_score" : 1.0,
        "_source" : {
          "title" : "title2",
          "centext" : "text2",
          "src" : ""
        }
      }
    ]
  }
}
**/ 
```

`took`:耗费了多少毫秒

`timed_out`:查询是否超时,可以手动指定超时时间。若执行时间到达超时时间，直接将已搜索到的数据返回 

`_shards`:本次搜索用到了多少个`shard(primary 或者replica)`

`hit.total`:一共搜索到多少条数据

`hit.max_score`:最大相关度，也就是查询出来的`document`对于查询的匹配度有多少

`hit.hits`:匹配的所有`document`,其中`_score`表示当前`document`的匹配度

`GET /web/article/_search?q=title:title2`查询`title`包含`title2`单词的`document`,可以在等号后面添加`+，-`，`+`表示必须包含这个单词，`-`表示必须不包含这个单。(`GET /web/article/_search?q=-title:title2`)

---

`query DSL(Domain Specified Language)`

将查询的各种要求放在`http requst body`中，格式是`json`格式

例：

```elixir
GET /web/article/_search
{
  "query": {
    "match": {
      "title": "title"
    }
  },
  "sort":[
    {"price":"desc"}
    ]
}
/**
	{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 2,
    "max_score" : null,
    "hits" : [
      {
        "_index" : "web",
        "_type" : "article",
        "_id" : "1",
        "_score" : null,
        "_source" : {
          "title" : "title",
          "centext" : "text",
          "price" : 4,
          "src" : ""
        },
        "sort" : [
          4
        ]
      },
      {
        "_index" : "web",
        "_type" : "article",
        "_id" : "3",
        "_score" : null,
        "_source" : {
          "title" : "title",
          "centext" : "text",
          "price" : 1,
          "src" : ""
        },
        "sort" : [
          1
        ]
      }
    ]
  }
}

**/
//符合语句查询，例如，以下查询是为了找出信件正文包含 business opportunity 的星标邮件，或者在收件箱正文包含 business opportunity 的非垃圾邮件：
{
    "bool": {
        "must": { "match":   { "email": "business opportunity" }},
        "should": [
            { "match":       { "starred": true }},
            { "bool": {
                "must":      { "match": { "folder": "inbox" }},
                "must_not":  { "match": { "spam": true }}
            }}
        ],
        "minimum_should_match": 1
    }
}
```

`sort`中的字段如果是`text`类型，需要`set filddata=true`,这会需要大量的内存空间，建议`sort`的字段类型是数字

`must`:必须包含

`should`:类似`or`，表示`"should":[A,B]=A or B`

`must_not`:不包含

---

`query filter`

```elixir
GET /web/article/_search
{
  "query": {
    "bool":{
      "must": {
        "match": {
         "title": "title2"
        }
      }, 
      "filter": {
        "range": {
          "price": {
            "gt": 4
          }
        }
      }
   }
  }
{
  "took" : 2,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "web",
        "_type" : "article",
        "_id" : "sad",
        "_score" : 0.2876821,
        "_source" : {
          "title" : "title2",
          "centext" : "text2",
          "price" : 5,
          "src" : ""
        }
      }
    ]
  }
}
```



---

`full-text search`:全文检索

```elixir
GET /web/article/_search
{
  "query": {
    "match": {
      "title": "title context"
    }
  }
}
```

将`title context `分别按`title,title context,context`来利用倒排索引进行检索，返回结果按匹配度排序，匹配度高的在前面。

---

`phrase search`：短语搜索

```elixir
GET /web/article/_search
{
  "query": {
    "match_phrase": {
      "title": "title context"
    }
  }
}
```

必须包含`title context`这个短语才能被匹配

---

`highlight search`：高亮搜索结果

```elixir
GET /web/article/_search
{
  "query": {
    "match_phrase": {
      "title": "title context"
    }
  },
"highlight": {
      "fields": {
        "title": {}
      }
    }
}
```

`fields.title`表示`title`这个字段要被高亮显示



​	