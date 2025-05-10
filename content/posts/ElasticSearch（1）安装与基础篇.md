---
title: ElasticSearch（1）安装与基础篇
date: 2024-07-11
draft: "false"
tags:
  - ES
---


### 一、安装es

①拉取镜像

②启动

```shell
docker run --name elasticsearch -p 9200:9200 -p 9300:9300 \
-e "discovery.type=single-node" \
-e ES_JAVA_OPTS="-Xms64m -Xmx128m" \
-v /mydata/elasticsearch/config/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml \
-v /mydata/elasticsearch/data:/usr/share/elasticsearch/data \
-v /mydata/elasticsearch/plugins:/usr/share/elasticsearch/plugins \
-d elasticsearch:8.11.3
```

③编写 elasticsearch.yml

```yaml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
xpack.security.enabled: false
```

> 1. `cluster.name: "docker-cluster"`
>    - 这个配置参数**定义了Elasticsearch集群的名称**。在分布式环境下，所有希望加入同一集群的节点都必须**配置相同的集群名称**。这里将集群命名为 "docker-cluster"，意味着所有带有这个配置的Elasticsearch节点将会组成一个名为“docker-cluster”的集群。
> 2. `network.host: 0.0.0.0`
>    - 此配置指定了Elasticsearch节点监听请求的网络接口地址。设置为 "0.0.0.0" 表示**节点将在所有可用网络接口上监听请求**，这意味着**该节点将对来自任何IP地址的客户端请求做出响应**。这对于Docker容器环境尤其常见，因为它**允许从宿主机或其他容器内部访问此Elasticsearch服务**。
> 3. `xpack.security.enabled: false`
>    - X-Pack是Elasticsearch提供的一个安全、监控、警报和图形化界面等扩展功能集合。`xpack.security.enabled` 参数控制X-Pack安全模块是否启用。当设置为 `false` 时，表示Elasticsearch节点启动时**不启用任何安全认证和授权机制**。这样做的结果是Elasticsearch服务将以无安全验证的方式运行，这在开发或测试环境中可能比较方便，但在生产环境中强烈建议启用并正确配置安全性以保护数据和系统不受未经授权的访问。



### 二、安装kibana

①拉取镜像

②启动

```shell
docker run --name kibana -e ELASTICSEARCH_HOSTS=http://192.168.200.134:9200 -p 5601:5601 \
-d kibana:8.11.3
```



### 三、使用elasticsearch

index 相当于数据库，type 相当于表

> Elasticsearch 是一个分布式、开源的搜索和分析引擎，主要用于全文搜索、日志数据分析、实时数据分析等应用。它是由 Elasticsearch N.V. 公司维护的开源项目，建立在 Apache Lucene 基础之上。以下是 Elasticsearch 的一些主要特性和用途：
>
> 1. **全文搜索：** Elasticsearch 提供强大的全文搜索功能，可以快速而准确地搜索大量文本数据。它使用倒排索引技术，支持复杂的查询和分析操作。
> 2. **分布式架构：** Elasticsearch 是一个分布式系统，可以横向扩展以处理大规模的数据。它将数据分散存储在多个节点上，实现了高可用性和容错性。
> 3. **实时数据分析：** Elasticsearch 支持实时数据索引和查询，适用于需要快速分析和可视化实时数据的场景，比如监控日志、事件数据等。
> 4. **多种数据类型支持：** Elasticsearch 不仅支持文本数据，还支持结构化数据、地理位置数据、数字数据等多种数据类型。这使得它在处理不同类型的数据时非常灵活。
> 5. **RESTful API：** Elasticsearch 提供基于 RESTful 风格的 API，使得与 Elasticsearch 的交互非常简单。可以使用 HTTP 方法（GET、POST、PUT、DELETE）进行索引、搜索、更新等操作。
> 6. **多语言支持：** Elasticsearch 提供了多种客户端，支持多种编程语言，如 Java、Python、Ruby、JavaScript 等，方便开发者在不同的语言环境中使用。
> 7. **聚合和分析：** Elasticsearch 支持聚合和分析操作，可以对搜索结果进行统计、分组、排序等操作，以便更深入地了解数据。
> 8. **开源和活跃社区：** Elasticsearch 是开源项目，拥有庞大的用户和开发者社区。社区的活跃性使得 Elasticsearch 得以不断更新和改进。
>
> Elasticsearch 在很多场景中被广泛应用，包括但不限于搜索引擎、日志和事件数据分析、业务智能、网络安全分析等领域。它的灵活性、高性能和易用性使得它成为处理大规模数据的首选工具之一。



### 四、es常用语法

Elasticsearch 是一个开源的分布式搜索和分析引擎，使用 JSON 对象来索引和查询数据。以下是一些常见的 Elasticsearch 查询和操作的示例：

1. **创建索引：**

```json
PUT /my_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2
  }
}
```

2. **插入文档：**

```json
POST /my_index/_doc/1
{
  "title": "Elasticsearch Basics",
  "content": "Introduction to Elasticsearch"
}
```

3. **检索文档：**

```json
GET /my_index/_doc/1
```

4. **搜索文档：**

- match_phrase：短语匹配

- multi_match：多字段匹配

  ```json
  # address 或 city 包含mill的都能匹配上
  GET bank/_search
  {
    "query": {
      "multi_match": {
        "query": "mill",
        "fields": ["address","city"]
      }
    }
  }
  ```

- bool：复合查询

  ```json
  GET bank/_search
  {
    "query": {
      "bool": {
        "must": [ # 必须匹配的条件
          {"match": {
              "gender": "M"
            }},
            {"match": {
              "address": "Mill"
            }}
        ],
        "must_not": [ # 必须不匹配的条件
          {"match": {
              "age": "18"
            }
          }
        ],
        "should": [ # 最好匹配的条件
          {"match": {
              "firstname": "Forbes"
            }}
        ]
      }
    }
  }
  ```

  ​

```json
GET /my_index/_search
{
  "query": { # 查询条件
    "match": {
      "title": "Elasticsearch Basics"
    }
  },
  "sort": [ # 排序条件
    {
      "balance":"desc"
    }
  ],
  "_source": ["balance","firstname"],  # 显示的字段
  "size": 10 # 显示的记录数
}
```

5. **聚合数据：**

```json
GET bank/_search
{
  "aggs":{
    "ageAgg":{ # 聚合名称
      "terms": { # 聚合类型
        "field": "age",
        "size": 5
      },
      "aggs":{  # 子聚合
        "balanceAvg2":{
          "avg": { 
            "field": "balance"
          }
        }
      }
    },
    "ageAvg":{
      "avg": {
        "field": "age"
      }
    },
    "balanceAvg":{
      "avg": {
        "field": "balance"
      }
    }
  }
}
```



```json
GET /my_index/_search
{
  "aggs": {
    "avg_content_length": {
      "avg": {
        "field": "content.length"
      }
    }
  }
}
```

6. **更新文档：**

```json
POST /my_index/_update/1
{
  "doc": {
    "content": "Advanced Elasticsearch Topics"
  }
}
```

7. 批量操作

```json
POST /my_index/_bulk
{ "index": { "_id": "1" } }
{ "title": "Document 1", "content": "Content for document 1" }
{ "index": { "_id": "2" } }
{ "title": "Document 2", "content": "Content for document 2" }
```

8. 映射

```json
# 查看映射
GET /bank/_mapping

# 创建索引并设置映射
PUT /my_index
{
  "mappings": {
    "properties": {
      "age":{"type": "integer"},
      "email":{"type":"keyword"},
      "name":{"type": "text"}
    }
  }
}

# 添加字段
PUT /my_index/_mapping
{
  "properties":{
    "employee_id":{
      "type":"keyword",
      "index":false
    }
  }
}
```

9. 数据迁移

无法修改已创建的字段类型，只能创建新的索引然后数据迁移。

```json
POST _reindex
{
  "source": {
    "index": "bank"
  },
  "dest":{
   	"index": "newbank"
  }
}
```



### 五、ik分词器搭建

### 六、RestHighLevelClient的使用（已经弃用）

①引入依赖

```xml
<dependency>
  <groupId>org.elasticsearch.client</groupId>
  <artifactId>elasticsearch-rest-high-level-client</artifactId>
  <version>7.17.16</version>
</dependency>
```

②配置类

```java
@Configuration
public class CommunityElasticsearchConfig {

    public static final RequestOptions COMMON_OPTIONS;

    static {
        RequestOptions.Builder builder = RequestOptions.DEFAULT.toBuilder();
        COMMON_OPTIONS = builder.build();
    }

    @Bean
    public RestHighLevelClient esRestClient(){
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(
                        new HttpHost("192.168.200.134", 9200, "http")));
        return client;
    }
}
```

③索引操作

```java
public void test2() throws IOException {
  		//插入数据/修改数据
        IndexRequest indexRequest = new IndexRequest("users");
        indexRequest.id("1");
  
        User user = new User();
        user.setAge(2);
        user.setUsername("文奇");
        user.setGender("女");
  
        String jsonString = JSON.toJSONString(user);
  		//json格式发送请求
        indexRequest.source(jsonString, XContentType.JSON);
		//执行，返回响应
        IndexResponse indexResponse = restHighLevelClient.index(indexRequest, CommunityElasticsearchConfig.COMMON_OPTIONS);

        System.out.println(indexResponse);

    }
```

④检索操作

```java
public void test3() throws IOException {
        //检索操作
        SearchRequest searchRequest = new SearchRequest();
        //检索的索引
        searchRequest.indices("bank");
        SearchSourceBuilder builder = new SearchSourceBuilder();
        //检索的条件
        builder.query(QueryBuilders.termQuery("address","mill"));
        //聚合
        builder.aggregation(AggregationBuilders.terms("ageAgg").field("age").size(10));
        builder.aggregation(AggregationBuilders.avg("balanceAvg").field("balance"));

        searchRequest.source(builder);
        //执行检索
        SearchResponse searchResponse = restHighLevelClient.search(searchRequest, CommunityElasticsearchConfig.COMMON_OPTIONS);

        //分析返回的结果
        SearchHits hits = searchResponse.getHits();
        SearchHit[] searchHits = hits.getHits();
        for (SearchHit hit : searchHits) {
            //封装为java对象
            String sourceAsString = hit.getSourceAsString();
            Account account = JSONObject.parseObject(sourceAsString, Account.class);
            System.out.println(account);
        }
        
        //获取检索的结果
        Aggregations aggregations = searchResponse.getAggregations();
        Terms ageAgg = aggregations.get("ageAgg");
        for (Terms.Bucket bucket : ageAgg.getBuckets()) {
            String keyAsString = bucket.getKeyAsString();
            System.out.println("ageAgg:" + keyAsString + ">>>>" + bucket.getDocCount());
        }

        Avg balanceAvg = aggregations.get("balanceAvg");
        System.out.println("balanceAvg:" + balanceAvg.getValue());
    }
```



### 六-2、ElasticsearchClient的使用（官方推荐）

#### ①依赖

这里选择版本的时候按照springboot内置的elasticsearch来选择。

```xml
<dependency>
  <groupId>org.elasticsearch</groupId>
  <artifactId>elasticsearch</artifactId>
  <version>8.7.1</version>
</dependency>

<dependency>
  <groupId>co.elastic.clients</groupId>
  <artifactId>elasticsearch-java</artifactId>
  <version>8.7.1</version>
</dependency>
```



#### ②配置

```java
import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.json.jackson.JacksonJsonpMapper;
import co.elastic.clients.transport.ElasticsearchTransport;
import co.elastic.clients.transport.rest_client.RestClientTransport;
import org.apache.http.HttpHost;
import org.elasticsearch.client.RestClient;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * @author vanky
 * @create 2024/1/20 11:45
 */
@Configuration
public class CommunityElasticsearchConfig {
    @Bean
    public ElasticsearchClient elasticsearchClient(){
        RestClient restClient = RestClient
                .builder(new HttpHost("192.168.200.134",9200)).build();

        ElasticsearchTransport transport = new RestClientTransport(
                restClient, new JacksonJsonpMapper());

        return new ElasticsearchClient(transport);
    }
}
```