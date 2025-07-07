### 聚合（Aggregations）
是一种强大的数据分析工具，用于对搜索结果进行统计、分组、计算等操作 。
### 分类
#### 1. 指标聚合（Metric Aggregations）
| 聚合类型          | 作用                          | 示例参数                                                            |
| ------------- | --------------------------- | --------------------------------------------------------------- |
| `avg`         | 计算字段的平均值                    | `{ "avg": { "field": "price" } }`                               |
| `sum`         | 计算字段的总和                     | `{ "sum": { "field": "quantity" } }`                            |
| `min/max`     | 计算字段的最小值 / 最大值              | `{ "min": { "field": "price" } }`                               |
| `stats`       | 一次性返回 count、min、max、avg、sum | `{ "stats": { "field": "price" } }`                             |
| `value_count` | 统计字段值的数量（去重后）               | `{ "value_count": { "field": "product" } }`                     |
| `percentiles` | 计算百分位数（如中位数、四分位数）           | `{ "percentiles": { "field": "price", "percents": [50, 95] } }` |

```json
PUT /sales_data
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "product": { "type": "keyword" },      // 产品名称
      "category": { "type": "keyword" },    // 类别
      "price": { "type": "double" },        // 价格
      "quantity": { "type": "integer" },    // 销售数量
      "date": { "type": "date" }            // 销售日期
    }
  }
}


POST /sales_data/_bulk
{"index":{"_id":1}}
{"product":"iPhone 14","category":"手机","price":7999,"quantity":100,"date":"2023-01-15"}
{"index":{"_id":2}}
{"product":"MacBook Pro","category":"电脑","price":12999,"quantity":50,"date":"2023-01-20"}
{"index":{"_id":3}}
{"product":"iPad Air","category":"平板","price":5999,"quantity":80,"date":"2023-01-22"}
{"index":{"_id":4}}
{"product":"iPhone 14 Pro","category":"手机","price":9999,"quantity":60,"date":"2023-01-25"}
{"index":{"_id":5}}
{"product":"MacBook Air","category":"电脑","price":9499,"quantity":70,"date":"2023-01-28"}

// 计算总销售额
GET /sales_data/_search
{
  "size": 0,  // 不返回文档，只返回聚合结果
  "aggs": {
    "total_revenue": {
      "sum": {
        "script": {
          "source": "doc['price'].value * doc['quantity'].value"  // 销售额 = 单价 × 数量
        }
      }
    }
  }
}
// 计算平均价格
GET /sales_data/_search
{
  "size": 0,
  "aggs": {
    "average_price": {
      "avg": { "field": "price" }
    }
  }
}
// 计算最高和最低价格
GET /sales_data/_search
{
  "size": 0,
  "aggs": {
    "max_price": { "max": { "field": "price" } },
    "min_price": { "min": { "field": "price" } }
  }
}
// 使用stats同时聚合多个指标
GET /sales_data/_search
{
  "size": 0,
  "aggs": {
    "price_stats": {
      "stats": { "field": "price" }
    }
  }
}
```

#### 2. 桶聚合（Bucket Aggregations）
桶聚合（Bucket Aggregations）是 Elasticsearch 中用于 **分组数据** 的强大工具，类似 SQL 中的 `GROUP BY`。它的核心逻辑是：
- **将文档根据特定规则分配到不同的 “桶”（Bucket）** 中，每个桶相当于一个分组。
- **支持嵌套**：桶内可以再嵌套桶或指标聚合，形成多层分析。
- **常见应用**：统计各类别的数量、按时间区间分组、筛选特定条件的数据等。

| 桶聚合方法            | 作用说明                         | 适用场景示例                                   | 核心参数                                                                                |
| ---------------- | ---------------------------- | ---------------------------------------- | ----------------------------------------------------------------------------------- |
| `terms`          | 按字段值分组，统计每个值的文档数量，默认取前 10 个值 | 统计不同品牌的商品数量、不同类别的订单数                     | `field`（分组字段）、`size`（返回分组数量）、`order`（排序方式，如按计数降序）                                   |
| `range`          | 按自定义数值范围分组，将字段值落入不同区间的文档归为一组 | 按价格区间（0-100、100-500 等）统计商品数量、按年龄范围统计用户分布 | `field`（数值字段）、`ranges`（定义区间，如`[{"to":100}, {"from":100, "to":500}]`）                |
| `date_range`     | 按自定义日期范围分组，对日期类型字段进行区间划分     | 按月份、季度统计订单量，按时间段（近 7 天、近 30 天）统计文章发布数    | `field`（日期字段）、`ranges`（日期区间，如`[{"to":"now-7d"}, {"from":"now-7d"}]`）、`format`（日期格式） |
| `histogram`      | 按固定间隔对数值字段分组，自动生成连续区间        | 按价格每 100 元为间隔统计商品分布、按年龄每 5 岁为间隔统计用户数量    | `field`（数值字段）、`interval`（间隔大小）、`min_doc_count`（最小文档数，过滤空桶）                          |
| `date_histogram` | 按固定时间间隔对日期字段分组（如天、周、月）       | 按天统计网站访问量、按月统计销售额                        | `field`（日期字段）、`calendar_interval`（日历间隔，如`day`/`month`）、`format`（日期格式）               |
| `filter`         | 按单个过滤条件创建一个桶，包含符合条件的所有文档     | 统计 “已付款” 状态的订单数、统计价格大于 1000 的商品数量        | `filter`（过滤条件，如`{"term": {"status": "paid"}}`）                                      |
| `filters`        | 按多个过滤条件创建多个桶，每个条件对应一个桶       | 同时统计 “已付款”“未付款”“已取消” 状态的订单数              | `filters`（键值对形式的多个过滤条件，如`{"paid": {"term": {"status": "paid"}}, "unpaid": {...}}`）  |
| `nested`         | 对嵌套类型字段进行聚合，可在嵌套文档内部使用其他聚合方法 | 统计商品嵌套评论中不同评分（1-5 星）的数量                  | `path`（嵌套字段路径，如`comments`）、`aggs`（嵌套内部的子聚合）                                         |


```json
DELETE /products

PUT /products
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 0
  },
  "mappings": {
    "properties": {
      "name": { "type": "keyword" },        // 产品名称
      "category": { "type": "keyword" },    // 类别
      "brand": { "type": "keyword" },       // 品牌
      "price": { "type": "double" },        // 价格
      "release_date": { "type": "date" }    // 发布日期
    }
  }
}

POST /products/_bulk
{"index":{"_id":1}}
{"name":"iPhone 14","category":"手机","brand":"Apple","price":7999,"release_date":"2022-09-16"}
{"index":{"_id":2}}
{"name":"Galaxy S23","category":"手机","brand":"Samsung","price":8999,"release_date":"2023-02-17"}
{"index":{"_id":3}}
{"name":"MacBook Pro","category":"笔记本","brand":"Apple","price":12999,"release_date":"2022-10-24"}
{"index":{"_id":4}}
{"name":"Surface Pro 9","category":"笔记本","brand":"Microsoft","price":9999,"release_date":"2022-10-27"}
{"index":{"_id":5}}
{"name":"iPad Air","category":"平板","brand":"Apple","price":5999,"release_date":"2022-03-18"}

// 执行 Terms 聚合：按品牌分组
GET /products/_search
{
  "size": 0,
  "aggs": {
    "brands": {
      "terms": {
        "field": "brand",
        "size": 10
      }
    }
  }
}

GET /products/_search

// 执行 Range 聚合：按价格区间分组
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          { "to": 6000 },         // 低价
          { "from": 6000, "to": 10000 },  // 中价
          { "from": 10000 }        // 高价
        ]
      }
    }
  }
}


// 嵌套聚合：按类别分组后计算平均价格
GET /products/_search
{
  "size": 0,
  "aggs": {
    "categories": {  // 第一层：按类别分组
      "terms": { "field": "category" },
      "aggs": {
        "avg_price": { "avg": { "field": "price" } }  // 第二层：计算每个类别的平均价格
      }
    }
  }
}

```
#### 3. 管道聚合（Pipeline Aggregations）
管道聚合是一种特殊的聚合类型，它 **不直接处理文档数据**，而是 **基于其他聚合的结果进行二次计算**。管道聚合可以让你对已有的聚合结果执行数学运算、排序、提取统计信息等操作。
