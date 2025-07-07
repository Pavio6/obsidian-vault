#### 相关性
**定义**：在搜索引擎中，描述一个文档与查询语句匹配程度的度量标准。Elasticsearch 使用评分机制（Scoring）来计算每个文档的相关性得分，得分高的文档排在前面。
##### TF-IDF算法（ES 5.x之前）
- **词频（Term Frequency, TF）**：词项在文档中出现的频率越高，得分越高
- **逆文档频率（Inverse Document Frequency, IDF）**：词项在所有文档中出现的频率越低，得分越高
- **字段长度归一化（Field-length Norm）**：字段越短，词项的权重越高
##### BM25算法（ES 5.x之后）
BM25 是 TF-IDF 的改进版本，解决了 TF-IDF 的一些缺陷：
- 对词频的饱和度处理更合理
- 对文档长度的处理更精细
- 可配置参数（k1 和 b）

#### 自定义评分
**定义**：自定义评分（Custom Scoring）允许你超越默认的相关性算法（如 BM25），根据业务需求定制文档的评分方式。


##### dis max & best fields、most fields、cross fields
##### Index Boost（索引级加权）
**定义**：
- 在多索引搜索时，可以给某些索引的文档设置更高的权重
- 来自高权重索引的文档会在结果中获得更高的相关性评分
- 不影响文档匹配，只影响排序结果
```json
// 官方新闻数据
PUT /official_news
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "content": { "type": "text" },
      "publish_date": { "type": "date" },
      "source": { "type": "keyword" }
    }
  }
}

// 第三方新闻数据  
PUT /third_party_news
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "content": { "type": "text" },
      "publish_date": { "type": "date" },
      "source": { "type": "keyword" }
    }
  }
}

// 插入官方新闻数据
POST /official_news/_bulk
{ "index": {} }
{ "title": "Elasticsearch 8.0 Released", "content": "Official release of Elasticsearch 8.0 with new features...", "publish_date": "2023-02-15", "source": "elastic" }
{ "index": {} }
{ "title": "Kibana Dashboard Improvements", "content": "New visualization tools in Kibana 8.0...", "publish_date": "2023-03-10", "source": "elastic" }

// 插入第三方新闻数据
POST /third_party_news/_bulk
{ "index": {} }
{ "title": "Elasticsearch Performance Tips", "content": "Third-party analysis of Elasticsearch performance...", "publish_date": "2023-03-05", "source": "tech_blog" }
{ "index": {} }
{ "title": "Comparing Elasticsearch and Solr", "content": "A detailed comparison between Elasticsearch and Apache Solr...", "publish_date": "2023-02-20", "source": "dev_portal" }


// 不使用index boost进行查询
GET /official_news,third_party_news/_search
{
  "query": {
    "match": { "content": "Elasticsearch" }
  }
}
// 使用index boost
// 结果为 official_news的_score会*2 third_party_news会*0.8
GET /official_news,third_party_news/_search
{
  "indices_boost": [
    { "official_news": 2.0 },  // 官方新闻权重加倍
    { "third_party_news": 0.8 }  // 第三方新闻权重降低
  ],
  "query": {
    "match": { "content": "Elasticsearch" }
  },
  "explain": false  // 可选：查看评分细节
}


// 创建带权重的别名
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "official_news",
        "alias": "high_priority_news"
      }
    },
    {
      "add": {
        "index": "third_party_news",
        "alias": "low_priority_news"
      }
    }
  ]
}

// 使用别名进行index boost查询
GET /high_priority_news,low_priority_news/_search
{
  "indices_boost": [ // 指定别名的权重
    { "high_priority_news": 3.0 },
    { "low_priority_news": 0.5 }
  ],
  "query": {
    "bool": {
      "must": [
        { "match": { "content": "Elasticsearch" } }
      ],
      "should": [
        { "match": { "title": "release" } }
      ]
    }
  }
}
```

##### Boosting
**定义**：Boost（提升）是 Elasticsearch 中调整查询或字段相关性的重要机制。
```json
DELETE /products
// 创建商品索引
PUT /products
{
  "mappings": {
    "properties": {
      "name": { 
        "type": "text",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": { "type": "text" },
      "price": { "type": "double" },
      "sales": { "type": "integer" },
      "is_premium": { "type": "boolean" },
      "tags": { "type": "keyword" }
    }
  }
}

// 插入商品数据
POST /products/_bulk
{ "index": { } }
{ "name": "Wireless Bluetooth Headphones", "description": "High-quality wireless headphones with noise cancellation", "price": 199.99, "sales": 1500, "is_premium": true, "tags": ["electronics", "audio"] }
{ "index": { } }
{ "name": "Smartphone X", "description": "Latest smartphone with 5G and 128GB storage", "price": 899.99, "sales": 800, "is_premium": true, "tags": ["electronics", "mobile"] }
{ "index": { } }
{ "name": "USB-C Cable", "description": "Durable 2m USB-C charging cable", "price": 12.99, "sales": 3000, "is_premium": false, "tags": ["accessories"] }
{ "index": { } }
{ "name": "Laptop Stand", "description": "Adjustable stand for laptops up to 17 inches", "price": 29.99, "sales": 1200, "is_premium": false, "tags": ["accessories", "workspace"] }


// 字段级别Boost
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "wireless",
      "fields": ["name^3", "description"]  // name字段权重是description的3倍
    }
  },
  "explain": true
}

GET /products/_search
// 查询级别Boost
GET /products/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "range": {
            "price": { "gt": 500 }
          },
          "boost": 2.0  // 高价商品权重加倍
        },
        {
          "match": { "name": "wireless" }
        }
      ]
    }
  }
}
```

##### Negative Boost（负向提升）
**定义**：Negative Boost（负向提升）是 Elasticsearch 中一种特殊的查询技术，它允许你**降低**（而非排除）某些文档的相关性分数，使它们排在结果列表的更后面，但不会完全过滤掉。

1. **与 must_not 的区别**：
    - `must_not`：完全排除匹配的文档
    - `negative_boost`：保留匹配的文档，但降低其分数
2. **典型应用场景**：
    - 降级显示过时内容
    - 降低低质量内容的排名
    - 减少但不完全排除某些类别的文档
    - A/B 测试中降低旧版本的显示优先级

```json
PUT /news_articles
{
  "mappings": {
    "properties": {
      "title": { "type": "text" },
      "content": { "type": "text" },
      "publish_date": { "type": "date" },
      "is_outdated": { "type": "boolean" },
      "category": { "type": "keyword" },
      "popularity": { "type": "integer" }
    }
  }
}

POST /news_articles/_bulk
{ "index": {} }
{ "title": "Elasticsearch 8.0 Released", "content": "Official announcement of the new version...", "publish_date": "2023-02-15", "is_outdated": false, "category": "technology", "popularity": 150 }
{ "index": {} }
{ "title": "Elasticsearch Basic Tutorial", "content": "Learn the basics of Elasticsearch...", "publish_date": "2021-05-10", "is_outdated": true, "category": "tutorial", "popularity": 300 }
{ "index": {} }
{ "title": "Advanced Search Techniques", "content": "Advanced methods for better search results...", "publish_date": "2023-01-20", "is_outdated": false, "category": "technology", "popularity": 200 }
{ "index": {} }
{ "title": "Elasticsearch 7.x EOL Notice", "content": "End of life announcement for version 7...", "publish_date": "2022-12-01", "is_outdated": true, "category": "announcement", "popularity": 100 }




// Negative Boost查询
// 降低过时文章的排名
GET /news_articles/_search
{
  "query": {
    "boosting": {
      "positive": { // 正向条件
        "match_all": {} // 匹配所有文档
      },
      "negative": {
        "term": {
          "is_outdated": {
            "value": "true" // 匹配过时文档
          }
        }
      },
      "negative_boost": 0.2 // 过时文档分数*0.2
    }
  }
}

GET /news_articles/_search

POST /news_articles/_bulk
{ "index": {} }
{ "title": "Elasticsearch for Beginners", "content": "Getting started with Elasticsearch", "publish_date": "2022-03-01", "is_outdated": false, "category": "tutorial", "popularity": 180 }
{ "index": {} }
{ "title": "Elasticsearch Performance Tuning", "content": "Optimize your Elasticsearch cluster", "publish_date": "2023-03-01", "is_outdated": false, "category": "technology", "popularity": 250 }
{ "index": {} }
{ "title": "Old Elasticsearch Version Overview", "content": "History of Elasticsearch versions", "publish_date": "2020-01-01", "is_outdated": true, "category": "tutorial", "popularity": 50 }

// 降低特定类别且流行度的内容
GET /news_articles/_search
{
  "query": {
    "boosting": {
      "positive": {
        "match": { "content": "Elasticsearch" }
      },
      "negative": {
        "bool": {
          "must": [
            { "term": { "category": "tutorial" } },
            { "range": { "popularity": { "lt": 200 } } }
          ]
        }
      },
      "negative_boost": 0.2
    }
  }
}

```

##### Function Score（函数评分）
**定义**：Function Score 是 Elasticsearch 中用于自定义文档评分规则的强大功能，它允许你修改或完全替换默认的相关性评分。
**主要功能**
- 修改查询返回文档的相关性评分
- 结合业务指标（如销量、点击量）影响排序
- 实现复杂的自定义排序逻辑
**核心参数**
- `query`：基础查询
- `functions`：评分函数列表
- `score_mode`：函数评分组合方式
- `boost_mode`：原始查询与函数评分组合方式
- `max_boost`：最大评分上限
- `min_score`：最低分数阈值

```json
DELETE /products


PUT /products
{
  "mappings": {
    "properties": {
      "name": { 
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "category": { "type": "keyword" },
      "price": { "type": "double" },
      "sales": { "type": "integer" },      // 销量
      "rating": { "type": "float" },       // 用户评分(1-5)
      "is_premium": { "type": "boolean" }, // 是否精品
      "stock": { "type": "integer" },      // 库存
      "created_at": { "type": "date" }     // 上架时间
    }
  }
}


POST /products/_bulk
{ "index": {} }
{ "name": "无线蓝牙耳机", "category": "电子产品", "price": 199.99, "sales": 1500, "rating": 4.5, "is_premium": true, "stock": 100, "created_at": "2023-01-15" }
{ "index": {} }
{ "name": "无线蓝牙耳机 Pro", "category": "电子产品", "price": 299.99, "sales": 800, "rating": 4.8, "is_premium": true, "stock": 50, "created_at": "2023-03-10" }
{ "index": {} }
{ "name": "有线耳机", "category": "电子产品", "price": 59.99, "sales": 3000, "rating": 4.2, "is_premium": false, "stock": 200, "created_at": "2022-11-20" }
{ "index": {} }
{ "name": "头戴式耳机", "category": "电子产品", "price": 159.99, "sales": 1200, "rating": 4.3, "is_premium": false, "stock": 80, "created_at": "2023-02-05" }
{ "index": {} }
{ "name": "运动蓝牙耳机", "category": "运动用品", "price": 129.99, "sales": 2500, "rating": 4.4, "is_premium": true, "stock": 150, "created_at": "2023-01-30" }
{ "index": {} }
{ "name": "游戏耳机", "category": "电子产品", "price": 89.99, "sales": 1800, "rating": 4.6, "is_premium": false, "stock": 60, "created_at": "2022-12-15" }
{ "index": {} }
{ "name": "降噪耳机", "category": "电子产品", "price": 249.99, "sales": 600, "rating": 4.7, "is_premium": true, "stock": 30, "created_at": "2023-03-01" }
{ "index": {} }
{ "name": "儿童耳机", "category": "电子产品", "price": 39.99, "sales": 900, "rating": 3.9, "is_premium": false, "stock": 120, "created_at": "2022-10-10" }
{ "index": {} }
{ "name": "耳机收纳盒", "category": "配件", "price": 19.99, "sales": 500, "rating": 4.0, "is_premium": false, "stock": 300, "created_at": "2022-09-01" }
{ "index": {} }
{ "name": "耳机清洁套装", "category": "配件", "price": 9.99, "sales": 700, "rating": 4.1, "is_premium": false, "stock": 250, "created_at": "2022-08-15" }

// 基础得分+销量加权
GET /products/_search
{
  "query": {
    "function_score": {
      "query": { 
        "match": { "name": "耳机" } 
      },
      "field_value_factor": { // 自定义评分函数
        "field": "sales",  // 字段
        "factor": 0.001, // 缩放因子
        "modifier": "none"
      },
      "boost_mode": "sum" // 得分合并方式，基础得分+函数得分
    }
  }
}
// 精品商品(is_premium=true)获得2倍权重
// 高评分商品获得更高分数
// 同时匹配"蓝牙"和"耳机"的商品分数更高
// "无线蓝牙耳机 Pro"(精品+高评分)会排名最前
GET /products/_search
{
  "query": {
    "function_score": {
      "query": { 
        "bool": {
          "should": [
            { "match": { "name": "蓝牙" } },
            { "match": { "name": "耳机" } }
          ]
        }
      },
      "functions": [
        {
          "filter": { "term": { "is_premium": true } },
          "weight": 2.0
        },
        {
          "field_value_factor": {
            "field": "rating",
            "factor": 1.5,
            "modifier": "log1p" // 对数变换
          }
        }
      ],
      "score_mode": "sum", // 多个函数间的组合方式，相加
      "boost_mode": "multiply" // 基础得分和函数得分的合并方式，相乘
    }
  },
  "explain": false
}
```

##### Rescore Query（重评分查询）
Rescore Query（重评分查询）是 Elasticsearch 提供的一种优化查询性能并改进结果相关性的技术，它允许对原始查询返回的顶部文档进行二次评分。
###### 什么是 Rescore
- 先执行主查询获取初步结果
- 然后只对前 N 个结果进行更复杂的二次评分
- 性能与精度的折中方案
###### 适用场景
- 主查询使用简单高效的查询
- 需要对顶部结果进行更精确的相关性计算
- 计算资源有限，无法对所有文档进行复杂评分

```json
DELETE /products

PUT /products
{
  "mappings": {
    "properties": {
      "name": { 
        "type": "text",
        "analyzer": "ik_max_word",
        "fields": {
          "keyword": { "type": "keyword" }
        }
      },
      "description": { "type": "text", "analyzer": "ik_max_word" },
      "price": { "type": "double" },
      "sales": { "type": "integer" },
      "rating": { "type": "float" },
      "tags": { "type": "keyword" },
      "created_at": { "type": "date" }
    }
  }
}

POST /products/_bulk
{ "index": {} }
{ "name": "无线蓝牙耳机", "description": "高品质无线蓝牙耳机，续航时间长", "price": 199.99, "sales": 1500, "rating": 4.5, "tags": ["电子", "音频"], "created_at": "2023-01-15" }
{ "index": {} }
{ "name": "智能手表", "description": "多功能健康监测智能手表", "price": 299.99, "sales": 800, "rating": 4.3, "tags": ["电子", "穿戴"], "created_at": "2023-03-10" }
{ "index": {} }
{ "name": "运动蓝牙耳机", "description": "防水防汗运动耳机", "price": 129.99, "sales": 2500, "rating": 4.4, "tags": ["运动", "音频"], "created_at": "2023-01-30" }
{ "index": {} }
{ "name": "平板电脑", "description": "高性能平板电脑", "price": 899.99, "sales": 400, "rating": 4.6, "tags": ["电子"], "created_at": "2023-02-05" }
{ "index": {} }
{ "name": "机械键盘", "description": "RGB背光机械键盘", "price": 89.99, "sales": 1800, "rating": 4.2, "tags": ["电子", "外设"], "created_at": "2022-12-15" }
{ "index": {} }
{ "name": "降噪耳机", "description": "主动降噪蓝牙耳机", "price": 249.99, "sales": 600, "rating": 4.7, "tags": ["电子", "音频"], "created_at": "2023-03-01" }
{ "index": {} }
{ "name": "电动牙刷", "description": "声波电动牙刷", "price": 39.99, "sales": 900, "rating": 4.0, "tags": ["个护"], "created_at": "2022-10-10" }
{ "index": {} }
{ "name": "移动电源", "description": "大容量快充移动电源", "price": 19.99, "sales": 500, "rating": 3.9, "tags": ["电子", "配件"], "created_at": "2022-09-01" }
{ "index": {} }
{ "name": "蓝牙音箱", "description": "便携式蓝牙音箱", "price": 79.99, "sales": 700, "rating": 4.1, "tags": ["电子", "音频"], "created_at": "2022-08-15" }
{ "index": {} }
{ "name": "无线充电器", "description": "15W快充无线充电器", "price": 29.99, "sales": 1200, "rating": 4.3, "tags": ["电子", "配件"], "created_at": "2022-11-20" }


// 基础重评分
GET /products/_search
{
  "query": { // 基础查询（粗排）
    "match": {
      "description": "蓝牙"
    }
  },
  "rescore": { // 二次评分（精排）
    "window_size": 5, // 窗口大小 对前5名进行二次评分
    "query": {
      "rescore_query": { // 精排查询
        "function_score": { // 评分函数
          "query": { "match_all": {} },
          "field_value_factor": { 
            "field": "rating",
            "factor": 1.2,
            "modifier": "log1p"
          },
          "boost_mode": "multiply"
        }
      },
      "query_weight": 0.7, // 基础查询权重
      "rescore_query_weight": 0.3 // 精排查询权重
    }
  }
}
```

#### 查询策略
##### dis_max
`dis_max` (Disjunction Max Query) 是 Elasticsearch 中一种多字段查询策略，它会执行多个子查询，但只取匹配度最高的子查询得分作为文档的最终得分（而非累加所有子查询得分）。

```json
DELETE /news_articles

PUT /news_articles
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0,
    "analysis": {
      "analyzer": {
        "my_ik_analyzer": {
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
        "analyzer": "my_ik_analyzer",  // 使用IK中文分词器             
        "fields": {
          "keyword": {
            "type": "keyword"           // 保留原始值用于精确匹配
          }
        }
      },
      "content": {
        "type": "text",
        "analyzer": "my_ik_analyzer",
        "copy_to": "full_text"          // 复制到组合字段
      },
      "author": {
        "type": "keyword"               // 作者字段不分词
      },
      "tags": {
        "type": "keyword",
        "null_value": "null"            // 处理空值
      },
      "publish_date": {
        "type": "date",
        "format": "yyyy-MM-dd||epoch_millis"
      },
      "popularity": {
        "type": "integer",
        "doc_values": true              // 优化聚合性能
      },
      "full_text": {                    // 组合字段
        "type": "text",
        "analyzer": "my_ik_analyzer"
      }
    }
  }
}



POST /news_articles/_bulk?refresh=true
{ "index": { "_id": "1" } }
{ "title": "Elasticsearch 8.0 新特性发布", "content": "Elasticsearch 8.0 带来了全新的安全功能和性能优化", "author": "张伟", "tags": ["技术", "搜索"], "publish_date": "2023-03-15", "popularity": 1500 }
{ "index": { "_id": "2" } }
{ "title": "大数据分析技术比较", "content": "对比 Elasticsearch 与 Hadoop 在数据分析领域的应用", "author": "李娜", "tags": ["大数据", "技术"], "publish_date": "2023-04-20", "popularity": 800 }
{ "index": { "_id": "3" } }
{ "title": "企业搜索解决方案", "content": "基于 Elasticsearch 构建的企业级搜索平台实战", "author": "王强", "tags": ["企业", "Elasticsearch"], "publish_date": "2023-05-10", "popularity": 1200 }
{ "index": { "_id": "4" } }
{ "title": "Elasticsearch 性能调优指南", "content": "深度解析 Elasticsearch 查询性能优化技巧", "author": "张伟", "tags": ["性能", "优化"], "publish_date": "2023-06-05", "popularity": 2000 }
{ "index": { "_id": "5" } }
{ "title": "搜索引擎技术发展史", "content": "从早期搜索引擎到现代 Elasticsearch 的技术演进", "author": "赵敏", "tags": ["历史", "搜索"], "publish_date": "2023-07-12", "popularity": 600 }
{ "index": { "_id": "6" } }
{ "title": "最新技术动态：Elasticsearch 8.1", "content": "Elasticsearch 8.1 即将推出的机器学习功能预览", "author": "李娜", "tags": ["新闻", "技术"], "publish_date": "2023-08-18", "popularity": 1800 }



// 去多个查询中的最高分作为最终得分
GET /news_articles/_search
{
  "query": {
    "dis_max": {
      "queries": [ // 并列的子查询列表
        {
          "match": {
            "title": {
              "query": "Elasticsearch 性能",
              "operator": "and",  // 必须同时包含两个词
              "boost": 3.0         // 标题匹配权重3倍
            }
          }
        },
        {
          "match": {
            "content": {
              "query": "Elasticsearch 性能",
              "minimum_should_match": "75%",  // 至少匹配75%的词项，上述的query也就是 2*0.75=1.5 
              "boost": 1.5
            }
          }
        },
        {
          "term": {
            "tags": {
              "value": "技术",
              "boost": 2.0
            }
          }
        }
      ],
      "tie_breaker": 0.3,  // 非最高分查询的贡献系数
      "boost": 1.5         // 整个dis_max查询权重
    }
  },
  "explain": true,  // 显示评分详情
  "highlight": {    // 高亮显示匹配内容
    "fields": {
      "title": {},
      "content": {}
    }
  }
}
```

