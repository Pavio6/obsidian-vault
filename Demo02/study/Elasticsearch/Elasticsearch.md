### ElasticSearch（ES）是一个开源的分布式搜索和数据分析引擎。能够达到近实时搜索。
#### 全文检索：从大量文本数据中快速检索出包含指定词汇或短语的信息的技术。
#### 倒排索引（Inverted Index）是一种核心的数据结构，它通过建立 “关键词到文档的映射关系”，实现对文档的快速检索。
**步骤：**
1. 文档预处理
2. 构建词典
3. 创建倒排列表
4. 存储索引文件
5. 查询处理

#### 索引（Index） - 类似于mysql中的表
**使用场景：**
- 将采集的不同业务类型的数据存储到不同的索引
-  按日期切分存储日志索引

**索引别名**是一个指向一个或多个索引的"虚拟名称"它可以像索引名称一样被用于查询、写入等操作，但提供了更高的灵活性和可维护性。

**相关代码**
```json
// 创建索引
PUT /student_index
{
  "settings": {
    "number_of_shards": 1, // 分片数
    "number_of_replicas": 1 // 副本数
  },
  // 指定mapping信息 
  "mappings": {
    "properties": {
      "name": {
        "type": "text"
      },
      "age": {
        "type": "integer"
      },
      "enrolled_date": {
        "type": "date"
      }
    }
  }
}
// 获取索引信息
GET /student_index


// 删除索引
DELETE /student_index

// 修改索引settings信息
PUT /student_index/_settings
{
  "index": {
    "number_of_replicas": 2
  }
}

// 修改索引mapping信息 - 增加一个字段
PUT /student_index/_mapping
{
  "properties": {
    "grade": {
      "type": "integer"
    }
  }
}

// 创建索引并指定别名
PUT myindex
{
  "aliases": {
    "myindex_aliases": {}
  }
}

GET myindex

GET myindex_aliases

// 为myindex索引添加一个别名 myindex_new
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "myindex",
        "alias": "myindex_new"
      }
    }
  ]
}

PUT tlmall_logs_202505
PUT tlmall_logs_202506
PUT tlmall_logs_202507
// 批量为所有添加别名
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "tlmall_logs_202505",
        "alias": "tlmall_logs_2025"
      }
    },
    {
      "add": {
        "index": "tlmall_logs_202506",
        "alias": "tlmall_logs_2025"
      }
    },
    {
      "add": {
        "index": "tlmall_logs_202507",
        "alias": "tlmall_logs_2025"
      }
    }
  ]
}

// 使用别名来查询索引信息
GET tlmall_logs_2025
```
#### 映射（Mapping） - 定义文档的结构以及如何对其进行索引和搜索
**静态映射（Explicit Mapping）**
**示例**
```json
PUT my_index 
{
	"mappings": {
		"properties": {
			"title": {"type": "text"},
			"author": {"type": "keyword"},
			"publish_date": {"type": "date"},
			"view": {"type": "long"},
			"tags": {"type": "keyword"},
			"location": {"type": "geo_point"}
		}
	}
}
```
#### 文档（Document） - 类似于一条数据

##### 新增文档
**批量新增时 需使用_bulk API**
```json
PUT /employee
// 新增文档 使用PUT 必须指定ID
PUT /employee/_doc/1
{
  "name": "张三",
  "sex": 1,
  "age": 25,
  "address": "广州天河公园",
  "remark": "golang developer"
}

// 新增文档 使用POST ID不是必须指定
POST /employee/_doc
{
  "name": "李四",
  "sex": 0,
  "age": 25,
  "address": "asdlkf",
  "remark": "alkf"
}

GET /employee/_search

PUT article
GET article/_search
// 批量新增
// Index 用于创建新文档或替换已有文档
// Create 如果文档不存在则创建 如果文档已存在则返回错误
// Update 更新现有文档
// Delete 删除指定文档

// 批量创建文档 使用create
POST _bulk
{"create":{"_index":"article","_id":3}}
{"id":3,"title":"fox老师"}
{"create":{"_index":"article","_id":4}}
{"id": 4,"title":"ksdaf"}

// 使用index 这里是替换已有文档
POST _bulk
{"index":{"_index":"article","_id":3}}
{"id":3,"title":"fox老师333"}
{"index":{"_index":"article","_id":4}}
{"id": 4,"title":"ksdaf333"}
```

##### 查询和删除文档
```json
DELETE /employee
// 创建索引并指定mappings和settings
PUT /employee
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword"
      },
      "sex": {
        "type": "integer"
      },
      "age": {
        "type": "integer"
      },
      "address": {
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "remark": {
        "type": "text",
        "analyzer": "ik_smart",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      }
    }
  }
}

// 批量新增
POST /employee/_bulk
{"index":{"_index":"employee","_id":"1"}}
{"name":"张三","sex":1,"age":25,"address":"广州天河公园","remark":"java developer"}
{"index":{"_index":"employee","_id":"2"}}
{"name":"李四","sex":1,"age":30,"address":"深圳南山科技园","remark":"python engineer"}
{"index":{"_index":"employee","_id":"3"}}
{"name":"王五","sex":0,"age":28,"address":"上海浦东软件园","remark":"frontend developer"}
{"index":{"_index":"employee","_id":"4"}}
{"name":"赵六","sex":1,"age":35,"address":"北京中关村","remark":"data analyst"}
{"index":{"_index":"employee","_id":"5"}}
{"name":"孙七","sex":0,"age":26,"address":"杭州西湖区","remark":"product manager"}
{"index":{"_index":"employee","_id":"6"}}
{"name":"周八","sex":1,"age":40,"address":"成都高新区","remark":"CTO"}


// 查询文档 指定ID
GET /employee/_doc/1

// 通过_mget可以一次查询多个id的记录
GET /employee/_mget
{
  "ids": ["1", "2"]
}

// 精确匹配 姓名为张三的文档
GET /employee/_search
{
  "query": {
    "term": {
      "name": {
        "value": "张三"
      }
    }
  }
}

POST _analyze
{
  "analyzer": "ik_max_word",
  "text": "广州白云山"
}
// 全文检索，查询在广州白云山的员工（搜索关键词）
GET /employee/_search
{
  "query": {
    "match": {
      "address": "广州白云山"
    }
  }
}

// 删除文档
DELETE /employee/_doc/1

GET /employee/_doc/1

// 批量删除
// POST /employee/_bulk 这里如果删除的文档索引都是一样的 
// 可以在请求路径中加上索引 下面的_index就不需要了
POST _bulk
{"delete":{"_index":"employee","_id":3}}
{"delete":{"_index":"employee","_id":4}}


GET /employee/_search
POST /employee/_bulk
{"index":{"_index":"employee","_id":"8"}}
{"name":"孙哈","sex":0,"age":26,"address":"杭州天空区","remark":"product manager"}
{"index":{"_index":"employee","_id":"9"}}
{"name":"周七","sex":1,"age":40,"address":"成都飞机区","remark":"CTO"}

# 删除在广州的员工
POST /employee/_delete_by_query
{
  "query": {
    "match": {
      "address": "杭州"
    }
  }
}
```

##### 更新文档
**更新行为**
- PUT请求在更新文档时会替换整个文档的内容，即使是文档中未更改的部分也会被新内容覆盖。
- POST请求在更新文档时可以使用_update API，这样可以在更新文档中的特定字段，而不是替换整个文档。
```json
GET /employee/_search

// 更新单个文档 使用_update
POST /employee/_update/2
{
  "doc": {
    "age": 288
  }
}

// 批量更新文档
POST /employee/_bulk
{"update":{"_id":2}}
{"doc":{"age":20002}}
{"update":{"_id":9}}
{"doc":{"age":20002}}

// 更新特定文档的某个字段
POST /employee/_update_by_query
{
  "query": {
    "term": { "name": "李四" }
  },
  "script": {
    "source": "ctx._source.put('age', 20)"
  }
}


GET /employee/_doc/9
// 并发场景下的更新文档
// 需指定if_seq_no和if_primary_term
POST /employee/_doc/9?if_seq_no=21&if_primary_term=1
{
  "name": "张三.....",
  "sex": 0
}

```

##### 练习
```json
PUT /product_info
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "productName": {
        "type": "text",
        "analyzer": "ik_smart"
      },
      "annual_rate": {
        "type": "keyword"
      },
      "describe": {
        "type": "text",
        "analyzer": "ik_smart"
      }
    }
  }
}

POST /product_info/_bulk
{"index":{"_index":"product_info"}}
{"productName":"理财产品A","annual_rate":"3.2200%","describe":"180天定期理财，最低20000起投，收益稳定，可以自助选择消息推送"}
{"index":{"_index":"product_info"}}
{"productName":"理财产品B","annual_rate":"3.5000%","describe":"365天定期理财，最低10000起投，适合长期投资，收益高于同类产品"}
{"index":{"_index":"product_info"}}
{"productName":"理财产品C","annual_rate":"2.8500%","describe":"90天短期理财，最低5000起投，灵活便捷，适合短期闲置资金"}
{"index":{"_index":"product_info"}}
{"productName":"理财产品D","annual_rate":"4.1000%","describe":"540天长期理财，最低50000起投，高收益高稳定性，适合风险偏好低的投资者"}
{"index":{"_index":"product_info"}}
{"productName":"理财产品E","annual_rate":"3.7500%","describe":"270天定期理财，最低15000起投，按月付息，适合追求稳定现金流的投资者"}
{"index":{"_index":"product_info"}}
{"productName":"理财产品F","annual_rate":"3.0000%","describe":"30天超短期理财，最低1000起投，灵活存取，收益高于活期存款"}

GET /product_info/_search

// 查询annual_rate在3.0和3.5之间的product
GET /product_info/_search
{
  "query": {
    "range": {
      "annual_rate": {
        "gte": "3.0000%",
        "lte": "3.5000%"
      }
    }
  }
}
```
---
### Elasticsearch中关联关系
##### 嵌套对象（Nested Object）

##### Join父子文档类型
##### 宽表冗余存储
##### 业务端关联

---
**[[Elasticsearch复杂查询]]**

**[[Elasticsearch搜索相关性]]**

**[[Elasticsearch聚合操作]]**

