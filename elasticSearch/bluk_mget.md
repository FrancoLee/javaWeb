## 批量操作

---

`_mget`:批量查询

```elixir
GET /_mget
{
  "docs":[{
    "_index":"web",
    "_type":"article",
    "_id":"1"
    
  },{
    "_index":"text",
    "_type":"t23",
    "_id":"1"
  }]
}
/**
{
  "docs" : [
    {
      "_index" : "web",
      "_type" : "article",
      "_id" : "1",
      "_version" : 1,
      "found" : true,
      "_source" : {
        "title" : "title",
        "centext" : "text",
        "price" : 4,
        "src" : ""
      }
    },
    {
      "_index" : "text",
      "_type" : "t23",
      "_id" : "1",
      "_version" : 1,
      "found" : true,
      "_source" : {
        "num" : 0,
        "str" : "s"
      }
    }
  ]
}
**/
```

`_bluk`:批量增删改(返回的数据会全部加载到内存中)

语法：

`{”action“:{"metdata"}}`

`{"data"}`



`action`:要执行的操作，有`4`种操作。

+ `delete`
+ `create` 等同`PUT /index/type/id/_create`强制创建
+ `index` 等同普通的`PUT`创建
+ `update` 执行`partial update`操作

`metdata`:要操作的数据的`_index,_type,_id`

`data`：要创建或则更新的对应数据

```elixir
POST _bulk
{"create":{"_index":"web","_type":"article","_id":"11"}}
{"title":"sdq","centext":"ss","price":"2"}
{"update":{"_index":"web","_type":"article","_id":"1"}}
{"doc":{"title":"lx"}}
{"delete":{"_index":"web","_type":"article","_id":"sad"}}
{"index":{ "_index":"text","_type":"t23", "_id":"3"}}
{"num":5,"str":"sta"}
/**
{
  "took" : 113,
  "errors" : false,
  "items" : [
    {
      "create" : {
        "_index" : "web",
        "_type" : "article",
        "_id" : "11",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 3,
        "_primary_term" : 5,
        "status" : 201
      }
    },
    {
      "update" : {
        "_index" : "web",
        "_type" : "article",
        "_id" : "1",
        "_version" : 2,
        "result" : "updated",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 5,
        "status" : 200
      }
    },
    {
      "delete" : {
        "_index" : "web",
        "_type" : "article",
        "_id" : "sad",
        "_version" : 2,
        "result" : "deleted",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 1,
        "_primary_term" : 5,
        "status" : 200
      }
    },
    {
      "index" : {
        "_index" : "text",
        "_type" : "t23",
        "_id" : "3",
        "_version" : 1,
        "result" : "created",
        "_shards" : {
          "total" : 2,
          "successful" : 1,
          "failed" : 0
        },
        "_seq_no" : 0,
        "_primary_term" : 2,
        "status" : 201
      }
    }
  ]
}
**/
```

