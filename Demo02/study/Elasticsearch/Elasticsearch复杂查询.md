#### match_all 查询
```json
// match_all 默认返回10条文档
// from：从哪条数据开始取
// size：取几条
GET /employee/_search
{
  "query": {
    "match_all": {}
  },
  "from": 3,
  "size": 3
}
// 根据age降序
GET /employee/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "age": "desc"
    }
  ],
  "from": 2,
  "size": 5
}
// _source 只返回哪些字段
GET /employee/_search
{
  "query": {
    "match_all": {}
  },
  "_source": ["name", "address"]
}
```

#### 精确匹配
**精确匹配指的是对搜索内容不经过文本分析直接用于文本匹配，搜索的对象大多是索引的非text类型字段**
##### 1. term单字段精确匹配查询
```json
GET /employee/_search

GET /employee

// 查询姓名为李四的员工信息
GET /employee/_search
{
  "query": {
    "term": {
      "name": {
        "value": "李四"
      }
    }
  }
}

// address.keyword 完整匹配
GET /employee/_search
{
  "query": {
    "term": {
      "address.keyword": {
        "value": "杭州余杭区未来科技城"
      }
    }
  }
}
// 如果只是address 它是text类型 那么匹配倒排索引中的数据
GET /employee/_search
{
  "query": {
    "term": {
      "address": {
        "value": "杭州"
      }
    }
  }
}
// 使用constant_score 查询
// 将查询转化为一个固定分数的过滤器（默认为1.0）
// 过滤执行效率高 适合频繁使用的查询条件
GET /employee/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "address.keyword": "杭州余杭区未来科技城"
        }
      }
    }
  }
}

POST /people/_bulk
{"index":{"_id":1}}
{"name": "小明", "interest": ["跑步", "篮球"]}
{"index":{"_id":2}}
{"name": "小红", "interest": ["跳舞", "画画"]}
{"index":{"_id":3}}
{"name": "小丽", "interest": ["跳舞", "唱歌", "跑步"]}

// tem处理多值字段（数组），term查询的是包含，也就是有就可以查询到
POST /people/_search
{
  "query": {
    "term": {
      "interest.keyword": {
        "value": "跑步"
      }
    }
  }
}
```
##### 2. terms多字段精确匹配
```json
POST /people/_search
{
  "query": {
    "terms": {
      "interest.keyword": [
        "画画",
        "跳舞"
      ]
    }
  }
}

```
##### 3. range范围查询
```json
GET /employee/_search

GET /employee
// 查询age在26-19之间的文档
GET /employee/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 26,
        "lte": 29
      }
    }
  }
}

```

##### 其他常用方法
```json
// name字段为多字段映射
// 主字段为text类型，用于全文搜索（分词处理）
// 子字段：keyword，用于精确匹配
PUT /products
{
  "mappings": {
    "properties": {
      "id": { "type": "keyword" },
      "name": { "type": "text", "fields": { "keyword": { "type": "keyword" } } }, 
      "category": { "type": "keyword" },
      "tags": { "type": "keyword" },
      "price": { "type": "double" },
      "description": { "type": "text" },
      "in_stock": { "type": "boolean" },
      "created_at": { "type": "date" }
    }
  }
}

POST /products/_bulk
{ "index": { "_id": "1" } }
{ "id": "P001", "name": "iPhone 14", "category": "手机", "tags": ["电子产品", "苹果", "智能手机"], "price": 7999, "in_stock": true, "created_at": "2023-09-15" }
{ "index": { "_id": "2" } }
{ "id": "P002", "name": "MacBook Pro", "category": "笔记本电脑", "tags": ["电子产品", "苹果", "电脑"], "price": 12999, "in_stock": true, "created_at": "2023-10-01" }
{ "index": { "_id": "3" } }
{ "id": "P003", "name": "华为 MateBook", "category": "笔记本电脑", "tags": ["电子产品", "华为", "电脑"], "price": 8999, "in_stock": false, "created_at": "2023-09-20" }
{ "index": { "_id": "4" } }
{ "id": "P004", "name": "小米手环7", "category": "智能穿戴", "tags": ["电子产品", "小米", "手环"], "price": 299, "in_stock": true, "created_at": "2023-08-10" }
{ "index": { "_id": "5" } }
{ "id": "P005", "name": "机械键盘 Cherry MX", "category": "外设", "tags": ["电脑配件", "键盘"], "price": 599, "in_stock": true, "created_at": "2023-11-05" }

POST /products/_doc
{
  "id": "M001",
  "name": "Huawei",
  "category": "手机",
  "tags": [
    "电子产品",
    "华为",
    "手机"
  ],
  "price": 13999,
  "in_stock": true,
  "created_at": "2025-06-10"
}


// exists：查询包含指定字段的文档
GET /products/_search
{
  "query": {
    "exists": {
      "field": "description"
    }
  }
}
// ids：查询id为2和4的文档
GET /products/_search
{
  "query": {
    "ids": {
      "values": ["2", "4"]
    }
  }
}

// prefix：匹配字段前缀
GET /products/_search
{
  "query": {
    "prefix": {
      "id": "P0"
    }
  }
}

// wildcard：支持通配符（* 匹配任意字符，? 匹配单个字符）
GET /products/_search
{
  "query": {
    "wildcard": {
      "name.keyword": "*Book*Pro"
    }
  }
}

// fuzzy查询：模糊匹配（支持编辑距离）
// 查询name近似iphon的产品 允许1个字符差异
GET /products/_search
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "iphon",
        "fuzziness": 1
      }
    }
  }
}
// Term Set 查询功能：匹配字段值包含指定集合中的任意元素
// 使用minimum_should_match指定 查询到的记录至少包含几个匹配项
// 查询 tags 中至少包含两个指定标签的产品
GET /products/_search
{
  "query": {
    "terms_set": {
      "tags": {
        "terms": ["电子产品", "苹果", "电脑"],
        "minimum_should_match": 2
      }
    }
  }
}

GET /products/_search
```

#### 全文检索
**基于相关性搜索和匹配文本数据，这些查询会对输入的文本进行分析，将其拆分成词项，并执行诸如分词、词干处理和标准化等操作。此类检索主要应用于非结构化文本数据，如文章和评论等。**

##### 1. match和multi_match
```json
DELETE /products

PUT /products
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "ik_smart_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart"
        },
        "ik_max_word_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "text",
        "analyzer": "ik_max_word_analyzer",
        "search_analyzer": "ik_smart_analyzer"
      },
      "description": {
        "type": "text",
        "analyzer": "ik_max_word_analyzer",
        "search_analyzer": "ik_smart_analyzer"
      },
      "category": {
        "type": "keyword"
      },
      "price": {
        "type": "float"
      },
      "tags": {
        "type": "text",
        "analyzer": "ik_smart_analyzer"
      }
    }
  }
}

POST /products/_bulk
{"index":{}}
{"name":"华为Mate40 Pro智能手机","description":"华为旗舰智能手机，搭载麒麟9000芯片，支持5G网络","category":"手机","price":5999,"tags":["华为","旗舰机","5G手机"]}
{"index":{}}
{"name":"小米11 Ultra旗舰手机","description":"小米高端智能手机，配备1亿像素相机和骁龙888处理器","category":"手机","price":5499,"tags":["小米","拍照手机","旗舰"]}
{"index":{}}
{"name":"Apple iPhone 13 Pro","description":"苹果最新智能手机，A15仿生芯片，超视网膜XDR显示屏","category":"手机","price":7999,"tags":["苹果","iOS","旗舰"]}
{"index":{}}
{"name":"华为MateBook X Pro笔记本","description":"华为旗舰笔记本电脑，13.9英寸3K触控全面屏","category":"笔记本","price":8999,"tags":["华为","轻薄本","触控屏"]}
{"index":{}}
{"name":"联想小新Pro16高性能笔记本","description":"联想高性能笔记本电脑，16英寸2.5K屏幕，标压处理器","category":"笔记本","price":6499,"tags":["联想","高性能","大屏"]} 

// match查询
// 它会对查询字符串进行分析，然后在指定的字段上执行全文检索。
// 会对查询文本进行分词处理（使用字段定义的分词器）
// IK分词器会将 "华为手机" 分词为 ["华为","手机"] 然后查询
// name包含这两个词之一的文档
GET /products/_search
{
  "query": {
    "match": {
      "name": "华为手机"
    }
  }
}

// 使用AND操作符：要求所有分词后的词元必须同时出现在文档中才会匹配。
// 会返回description中包含"旗舰" "手机" 的文档
// 这里的query中的值 使用空格分割 代表需要拆分词
// 也就是说会将 "旗舰 手机" 拆分成 "旗舰" "手机"
GET /products/_search
{
  "query": {
    "match": {
      "description": {
        "query": "旗舰 手机",
        "operator": "and"
      }
    }
  }
}
// multi_match查询是match查询的多字段版本，
// 允许在多个字段上执行相同的match查询。
// 如下：将在name、description、tags中查询有"旗舰"的文档
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "旗舰",
      "fields": ["name", "description", "tags"]
    }
  }
}
// 字段提升
// 如下 name的字段权重最高
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "华为",
      "fields": ["name^3", "description^2", "tags"] 
    }
  }
}
```

##### 2. mach_phrase、query_string、simple_query_string
```json
// match_phrase 查询用于精确匹配包含指定词项序列的文档，词项必须：
// 全部出现在字段中 按照查询中的顺序出现 位置相邻

GET /products/_search
{
  "query": {
    "match_phrase": {
      "description": "旗舰智能手机"  // 必须包含连续的"旗舰 智能 手机"（按IK分词结果）
    }
  }
}

GET /products/_search
{
  "query": {
    "match_phrase": {
      "description": {
        "query": "华为 笔记本",  // 这两个词不必相邻
        "slop": 5  // 允许最多5个其他词项间隔
      }
    }
  }
}

GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "华为 旗舰",
      "type": "phrase",  // 相当于多字段的match_phrase
      "fields": ["name", "description"]
    }
  }
}



// query_string 查询 - 支持与或非表达式的查询
GET /_search
{
  "query": {
    "query_string": {
      "query": "关键词1 AND (关键词2 OR 关键词3) NOT 关键词4",
      "default_field": "content",  // 默认搜索字段
      "default_operator": "OR"     // 默认操作符（当无显式操作符时）
    }
  }
}
GET /products/_search

// 基础查询
GET /products/_search
{
  "query": {
    "query_string": {
      "default_field": "description",  // 默认搜索字段
      "query": "旗舰 AND 手机"  // 必须同时包含"旗舰"和"手机"
    }
  }
}

GET /products/_search
{
  "query": {
    "query_string": {
      "default_field": "price",
      "query": "price: [8000 TO 10000]"
    }
  }
}

// 多字段搜索
GET /products/_search
{
  "query": {
    "query_string": {
      "fields": ["name", "description"],  // 搜索多个字段
      "query": "(华为 OR 小米) AND 手机"  // 华为或小米的手机
    }
  }
}

// 使用字段限定和通配符
// 查询 名字以"华"开头且价格在5000-8000之间
GET /products/_search
{
  "query": {
    "query_string": {
      "query": "name:华* AND price:[5000 TO 8000]"
    }
  }
}

// 短语搜索
GET /products/_search
{
  "query": {
    "query_string": {
      "default_field": "description",
      "query": "\"麒麟9000芯片\""  // 精确匹配整个短语
    }
  }
}

// 模糊搜索
GET /products/_search
{
  "query": {
    "query_string": {
      "default_field": "name",
      "query": "华为~"  // 搜索"华为"的近似词（模糊匹配）
    }
  }
}





// simple_query_string 查询
GET /_search
{
  "query": {
    "simple_query_string": {
      "query": "关键词1 +关键词2 -关键词3",
      "fields": ["title", "content^2"],  // 搜索多个字段（支持权重）
      "default_operator": "OR"          // 默认操作符
    }
  }
}

// 基础查询
// 在 name 或 description 字段中，必须同时包含"旗舰"和"手机"的文档
GET /products/_search
{
  "query": {
    "simple_query_string": {
      "query": "旗舰 +手机",  // 必须包含"旗舰"和"手机"
      "fields": ["name", "description"]
    }
  }
}

```

**精确匹配与全文检索的本质区别为：**
精确不对带检索文本进行分词，全文检索会分词，然后对每个词条进行单独检索。

#### bool query布尔查询

##### must、should、must_not、filter、bool代码示例
```json
GET /products/_search
// must查询 - 必须满足所有条件
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "description": "旗舰"  // 描述中必须包含"旗舰"（使用IK分词器分词）
          }
        },
        {
          "term": {
            "category": "手机"  // 类别必须精确匹配"手机"（keyword类型字段）
          }
        }
      ]
    }
  }
}


// should 查询（加分条件）
// 默认情况下，满足的should条件越多，文档评分越高
// minimum_should_match 默认是0（可以不满足任何should条件）
GET /products/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "range": {
            "price": {
              "gte": 5000  // 价格≥5000的文档会获得更高评分
            }
          }
        },
        {
          "match": {
            "tags": "5G手机"  // 标签包含"5G手机"的文档会获得更高评分
          }
        }
      ],
      "minimum_should_match": 1  // 至少满足一个should条件
    }
  }
}


// must_not 查询（排除条件）
// 只要文档不满足 must_not 中的任何一个条件，就会被保留。
GET /products/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "term": {
            "name.keyword": "Apple iPhone 13 Pro"  // 排除名为"Apple iPhone 13 Pro"的文档
          }
        },
        {
          "range": {
            "price": {
              "gt": 8000  // 排除价格>8000的文档
            }
          }
        }
      ]
    }
  }
}

GET /products/_search

// filter 查询 
// filter条件不参与评分，但会被缓存，性能更好
// exists确保字段存在 range过滤price区间
GET /products/_search
{
  "query": {
    "bool": {
      "filter": [
        {
          "exists": {
            "field": "tags"  // 必须存在tags字段
          }
        },
        {
          "range": {
            "price": {
              "gte": 3000,
              "lte": 7000
            }
          }
        }
      ]
    }
  }
}



// bool 组合查询
// 执行顺序为
// 1. filter和must_not子句
// 2. must子句
// 3. should子句
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "description": "旗舰"
          }
        },
        {
          "term": {
            "category": "手机"
          }
        }
      ],
      "should": [
        {
          "range": {
            "price": {
              "gte": 5000
            }
          }
        },
        {
          "match": {
            "tags": "5G手机"
          }
        }
      ],
      "must_not": [
        {
          "term": {
            "name.keyword": "Apple iPhone 13 Pro"
          }
        },
        {
          "range": {
            "price": {
              "gt": 8000
            }
          }
        }
      ],
      "filter": [
        {
          "exists": {
            "field": "tags"
          }
        },
        {
          "range": {
            "price": {
              "gte": 3000,
              "lte": 7000
            }
          }
        }
      ],
      "minimum_should_match": 1
    }
  },
  "from": 0,
  "size": 10,
  "sort": [
    {
      "_score": "desc"
    },
    {
      "price": "asc"
    }
  ]
}
```

#### Highlighting（高亮显示）
用于在搜索结果中标记出匹配查询条件的文本片段，帮助用户快速定位到文档中的相关部分。
##### 概念
- **高亮字段**：指定哪些字段需要高亮显示
- **高亮标签**：用于标记匹配文本的HTML标签（默认是`<em>`）
- **片段**：从原始文本中提取的包含匹配内容的部分文本
- **片段大小**：控制每个片段的字符数
##### 工作原理
1. 首先执行查询找到匹配的文档
2. 对指定字段重新分析并定位匹配的词项
3. 提取包含匹配内容的文本片段
4. 用指定的标签包裹匹配的词项
##### 代码示例
```json
GET /products/_search
{
  "query": {
    "match": {
      "description": "旗舰"
    }
  },
  "highlight": {
    "fields": {
      "description": {} // 对 description字段高亮
    }
  }
}


// 多字段高亮
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "旗舰",
      "fields": ["name", "description"]
    }
  },
  "highlight": {
    "fields": {
      "name": {},
      "description": {}
    }
  }
}

// 短语查询高亮
GET /products/_search
{
  "query": {
    "match_phrase": {
      "description": "旗舰智能手机"
    }
  },
  "highlight": {
    "fields": {
      "description": {
        "type": "plain"  // 使用plain高亮器
      }
    }
  }
}
```

#### 练习：CSDN博客文章搜索
```json
// 创建索引和映射
PUT /csdnblogs
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "ik_smart_analyzer": {
          "type": "custom",
          "tokenizer": "ik_smart"
        },
        "ik_max_word_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word_analyzer",
        "search_analyzer": "ik_smart_analyzer"
      },
      "content": {
        "type": "text",
        "analyzer": "ik_max_word_analyzer",
        "search_analyzer": "ik_smart_analyzer"
      },
      "author": {
        "type": "keyword",
        "fields": {
          "text": {
            "type": "text",
            "analyzer": "ik_smart_analyzer"
          }
        }
      },
      "tags": {
        "type": "keyword",
        "fields": {
          "text": {
            "type": "text",
            "analyzer": "ik_smart_analyzer"
          }
        }
      },
      "date": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "view_count": {
        "type": "integer"
      },
      "like_count": {
        "type": "integer"
      }
    }
  }
}

// 插入数据
POST /csdnblogs/_bulk
{"index":{}}
{"title":"Elasticsearch入门教程","content":"本文介绍Elasticsearch的基本概念和安装方法","author":"搜索专家","tags":["Elasticsearch","教程","入门"],"date":"2023-05-15","view_count":1500,"like_count":120}
{"index":{}}
{"title":"Python数据分析实战","content":"使用Python Pandas进行数据分析的完整指南","author":"Python大神","tags":["Python","数据分析","Pandas"],"date":"2023-06-20","view_count":3200,"like_count":250}
{"index":{}}
{"title":"Java高并发编程","content":"深入讲解Java并发编程的核心技术","author":"Java架构师","tags":["Java","并发编程","多线程"],"date":"2023-04-10","view_count":2800,"like_count":180}
{"index":{}}
{"title":"Elasticsearch性能优化","content":"Elasticsearch集群性能优化的10个技巧","author":"搜索专家","tags":["Elasticsearch","性能优化"],"date":"2023-07-05","view_count":4200,"like_count":350}
{"index":{}}
{"title":"Spring Boot实战","content":"Spring Boot企业级应用开发指南","author":"Java架构师","tags":["Java","Spring Boot"],"date":"2023-03-12","view_count":1900,"like_count":90}

// 全文搜索
// 在标题、内容和标签中搜索"Elasticsearch 教程"
// 标题权重更高(^3)
// 对标题和内容进行高亮显示
// 按相关度和浏览量排序
GET /csdnblogs/_search
{
  "query": {
    "multi_match": {
      "query": "Elasticsearch 教程",
      "fields": ["title^3", "content", "tags.text"], 
      "type": "best_fields" // 取得分最高的字段作为最终评分
    }
  },
  "highlight": {
    "fields": {
      "title": {}, // 高亮标题(默认配置)
      "content": { // 高亮内容(自定义配置)
        "fragment_size": 100, // 每个高亮片段长度为100个字符
        "number_of_fragments": 2 // 返回2个片段
      }
    },
    "pre_tags": ["<em class='highlight'>"], // 高亮前缀标签(用于CSS)
    "post_tags": ["</em>"] // 高亮后缀标签
  },
  "sort": [
    {
      "_score": "desc" // 按相关性评分降序
    },
    {
      "view_count": "desc" // // 评分相同时，按浏览量降序排列
    }
  ],
  "from": 0,
  "size": 10
}

// 高级布尔搜索
GET /csdnblogs/_search
{
  "query": {
    "bool": {
      "must": [ // 必须满足的条件
        {
          "match": {
            "content": "编程"
          }
        }
      ],
      "should": [ // 至少满足一个的条件
        {
          "term": {
            "tags": "Java"
          }
        },
        {
          "range": {
            "like_count": {
              "gte": 100
            }
          }
        }
      ],
      "must_not": [ // 必须不满足的条件
        {
          "term": {
            "author": "Python大神"
          }
        }
      ],
      "filter": [ // 过滤条件(不影响评分)
        {
          "range": {
            "date": {
              "gte": "2023-01-01",
              "lte": "2023-12-31"
            }
          }
        }
      ],
      "minimum_should_match": 1 // 至少需要匹配一个should条件
    }
  }
}

// 作者搜索+标签过滤
GET /csdnblogs/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "match": {
            "author.text": "架构师"
          }
        }
      ],
      "filter": [
        {
          "terms": {
            "tags": ["Java", "Spring Boot"]
          }
        }
      ]
    }
  },
  "sort": [
    {
      "date": "desc"
    }
  ]
}
```



