## mapping

---

`excat value`:精准匹配，比如：`q=field_name:field_value`。

`full  text`:全文检索，比如：`q=value`。

区别：`full text`对比`excat value`来说用的分词器不同。`full text`是会做`normalization(大小写，同义词，符号消除等)`。比如日期`2018-12-31`，对于`excat value`来说分词后是`2018-12-31`,搜索`2018`并不会搜索到(`5.2`版本后有所优化，能搜索到一个结果)，而对于`full text`来说会是`2018`,`12`,`31`，`2018`、`12`或者`31`都能搜到。

分词过程：

+ `character filter`:预处理，去除无用信息(`html标签`)，特殊符号转换`&->and`等
+ `tokenizer`:分词，`i love you`->`i`,`love`,`you`
+ `token filter`:标准化，去除无用单词(`a/an/the`),转换成小写，转化同义词等。

内置分词器：

+ `standard`分词器：（默认的）它将词汇单元转换成小写形式，并去掉停用词（`a`、`an`、`the`等没有实际意义的词）和标点符号，支持中文采用的方法为单字切分（例如，`你好`切分为`你`和`好`）。
+ `simple`分词器：首先通过非字母字符来分割文本信息，然后将词汇单元同一为小写形式。该分析器会去掉数字类型的字符。
+ `Whitespace`分词器：仅仅是去除空格，对字符没有`lowcase`（大小写转换）化，不支持中文；并且不对生成的词汇单元进行其他的标准化处理。
+ `language`分词器：特定语言的分词器，不支持中文。

```
//测试分词器
GET /_analyze
{
  "analyzer": "standard",
  "text": ["清华大学"]
}
/** standard对中文使用单字拆分
{
  "tokens" : [
    {
      "token" : "清",
      "start_offset" : 0,
      "end_offset" : 1,
      "type" : "<IDEOGRAPHIC>",
      "position" : 0
    },
    {
      "token" : "华",
      "start_offset" : 1,
      "end_offset" : 2,
      "type" : "<IDEOGRAPHIC>",
      "position" : 1
    },
    {
      "token" : "大",
      "start_offset" : 2,
      "end_offset" : 3,
      "type" : "<IDEOGRAPHIC>",
      "position" : 2
    },
    {
      "token" : "学",
      "start_offset" : 3,
      "end_offset" : 4,
      "type" : "<IDEOGRAPHIC>",
      "position" : 3
    }
  ]
}**/

```

`mapping`:

​	`elasticsearch`中`field`的一些特殊属性(用什么分词器，是否分词等)。`es`会自动识别数据类型，也可以手动自定`mapping`.自动识别是在第一次添加这个字段时识别。

​	已有数据的`mapping`不能修改，只能新增。

​	核心数据类型：`boolean,long,double,date,string/text`(`text`会分词，支持模糊和精确查询，不支持聚合查询。`keyword`不会进行分词，支持聚合)

​	复杂类型

+ `multivalue field`：一个字段里可以有多个同类型的数据，例：`{"OJ":["POJ",'ZOJ']}`
+ `empty field`:空字段`null,[],[null]`
+ `object`:字段嵌套字段

​	`GET /index/_mapping/type`:查看索引的`mapping`信息

```
//新建字段并设置索引信息
PUT /school
{
  "mappings": {
    "schools":{
      "properties":{
        "school":{
          "type":"text",
          "analyzer":"ik_max_word"
        }
      }
    }
  }
}
//新增字段并设置mapping信息
PUT /web/_mapping/article/
{
    "properties":{
      "school":{
        "type":"text",
         "analyzer":"ik_max_word"
      }
     }
}
//获取索引的mapping信息
GET /web/article/_mapping
```



