---
title: ElasticSearch（2）使用篇
date: 2024-07-14
draft: "false"
tags:
  - ES
---


## 一、创建对象映射

①创建ElasticSearch中对象

> 不参与索引的可以加上
>
> ```json
> "index": false,
> "doc_values": false
> ```



```json
PUT article
{
  "mappings": {
    "properties": {
      "id":{
        "type": "long",
        "index": false,
        "doc_values": false
      },
      "title":{
        "type": "text",
        "analyzer": "ik_smart"
      },
      "createTime":{
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss"
      },
      "images":{
        "type": "keyword",
        "index": false,
        "doc_values": false
      }
    }
  }
}
```



②java对象映射

**注意LocalDateTime的处理。**

```java
@Data
public class ArticleEsModel {

    /**
     * id
     */
    private Long id;

    /**
     * 标题
     */
    private String title;
  
  	/**
     * 时间
     */
    @Field(type = FieldType.Date,format = DateFormat.basic_date_time_no_millis,pattern = "yyyy-MM-dd HH:mm:ss")
    @JsonSerialize(using = LocalDateTimeSerializer.class)
    @JsonDeserialize(using = LocalDateTimeDeserializer.class)
    @JsonFormat(shape = JsonFormat.Shape.STRING,timezone = "GMT+8", pattern = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createTime;

    /**
     * 图片
     */
    private List<String> images;
}
```



## 二、批量插入

```java
import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch.core.BulkRequest;
import co.elastic.clients.elasticsearch.core.BulkResponse;
import co.elastic.clients.elasticsearch.core.bulk.BulkResponseItem;
import com.vanky.community.api.search.to.ArticleEsModel;
import com.vanky.community.search.constant.EsContent;
import com.vanky.community.search.service.ArticleSearchService;
import jakarta.annotation.Resource;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

/**
 * @author vanky
 */
@Slf4j
@Service
public class ArticleSearchServiceImpl implements ArticleSearchService {

    @Resource
    private ElasticsearchClient elasticsearchClient;

    @Override
    public boolean saveArticleBatch(List<ArticleEsModel> articleEsModels) throws IOException {
        BulkRequest.Builder br = new BulkRequest.Builder();

        for (ArticleEsModel model : articleEsModels) {
            br.operations(op -> op
                    .index(idx -> idx
                            .index(EsContent.ARTICLE_INDEX)
                            .id(model.getId().toString())
                            .document(model)
                    )
            );
        }
        //执行操作,并返回响应结果
        BulkResponse bulkResponse = elasticsearchClient.bulk(br.build());
        
        //处理错误
        if(bulkResponse.errors()){
            List<String> collect = Arrays.stream(bulkResponse.items().toArray(new BulkResponseItem[0]))
                    .map(item -> item.id())
                    .collect(Collectors.toList());
            
            log.error("文章保存到es出错：{}", collect);
            return false;
        }

        return true;
    }
}
```



## 三、查询

ElasticSearch  ---- DSL查询语句

需要包含：**模糊匹配，过滤，排序，分页，高亮，聚合分析**

```json
//GET xxx/_search
{
  "query":{
    "bool":{
      "must":[],
      "filter":[]
    }
  },
  "sort":[],
  "from":0,
  "size":1,
  "highlight":{
    "fields":{"title":{}},
    "pre_tags":"<b style='color:red'>",
    "post_tags":"</b>"
  },
  "aggs":{}
}
```



### java中执行查询

1. 检索基本逻辑

```java
@Override
public SearchResult<WorkSearchVo> searchWithKeyword(SearchParam searchParam) {
  //1.构建查询请求
  SearchRequest searchRequest = buildRequest(searchParam);

  SearchResult<WorkSearchVo> searchResult = null;
  try {
    //2.执行查询请求，获取查询结果
    SearchResponse<WorkSearchVo> response = elasticsearchClient.search(searchRequest, WorkSearchVo.class);
    //3.整理查询结果
    searchResult = buildSearchResult(response, searchParam);
  } catch (IOException e) {
    log.error("检索失败！检索参数：{}", searchParam);
    throw new RuntimeException(e);
  }

  return searchResult;
}
```

2. 构建检索请求方法

```java
public SearchRequest buildRequest(SearchParam searchParam){
  SearchRequest.Builder requestBuilder = new SearchRequest.Builder();
  requestBuilder.index("works");

  //Query
  Query.Builder query = new Query.Builder();
  BoolQuery.Builder bool = new BoolQuery.Builder();

  //1. 关键字匹配
  if(StringUtils.hasText(searchParam.getKeyword())){
    bool.should(s -> s
                .match(m -> m
                       .field("workTitle")
                       .query(searchParam.getKeyword())));

    bool.should(s -> s
                .match(m -> m
                       .field("content")
                       .query(searchParam.getKeyword())));
  }

  //2. 过滤
  //2.1 校区
  if (searchParam.getCampus() != 0){
    bool.filter(f -> f.term(t -> t.field("campus")
                            .value(searchParam.getCampus())));
  }
  //2.2 模块
  if(searchParam.getModule() != 0){
    bool.filter(f -> f.term(t -> t.field("module")
                            .value(searchParam.getModule())));
  }
  //2.3 类型
  if (searchParam.getType() != TypeEnum.SearchType.ALL.getValue()){
    bool.filter(f -> f.term(t -> t.field("type")
                            .value(searchParam.getType())));
  }

  query.bool(bool.build());

  requestBuilder.query(query.build());

  //排序条件
  //sort
  if (searchParam.getSort() == TypeEnum.SortType.TIME_DESC.getValue()){
    requestBuilder.sort(s -> s
                        .field(f -> f
                               .field("createTime")
                               .order(SortOrder.Desc))
                       );
  } else if (searchParam.getSort() == TypeEnum.SortType.TIME_ASC.getValue()) {
    requestBuilder.sort(s -> s
                        .field(f -> f
                               .field("createTime")
                               .order(SortOrder.Asc))
                       );
  }

  //高亮查询
  if(!searchParam.getKeyword().isBlank()){
    HashMap<String, HighlightField> map = new HashMap<>();

    map.put("workTitle", new HighlightField.Builder().matchedFields("workTitle").build());
    map.put("content", new HighlightField.Builder().matchedFields("content").build());

    requestBuilder.highlight(h -> h
                             .preTags("<b style='color:red'>")
                             .postTags("</b>")
                             .fields(map));
  }

  //分页查询
  requestBuilder.from((searchParam.getPageNum() - 1) * EsContent.PAGE_SIZES);
  requestBuilder.size(EsContent.PAGE_SIZES);

  SearchRequest searchRequest = requestBuilder.build();

  return searchRequest;
}
```

3. 整理检索结果的方法，构建返回给客户端的结果

```java
public SearchResult<WorkSearchVo> buildSearchResult(SearchResponse<WorkSearchVo> searchResponse, SearchParam searchParam){
  SearchResult<WorkSearchVo> result = new SearchResult<>();

  //获取记录
  List<WorkSearchVo> searchVos = new ArrayList<>();
  for (Hit<WorkSearchVo> hit : searchResponse.hits().hits()) {
    WorkSearchVo searchVo = hit.source();
    //改写高亮部分内容
    if (StringUtils.hasText(searchParam.getKeyword())){
      if (hit.highlight().get("content") != null){
        searchVo.setContent(hit.highlight().get("content").get(0));
      }

      if (hit.highlight().get("workTitle") != null){
        searchVo.setWorkTitle(hit.highlight().get("workTitle").get(0));
      }
    }

    searchVos.add(searchVo);
  }
  result.setObjects(searchVos);

  //记录数
  Long total = searchResponse.hits().total().value();
  result.setTotal(total);

  //页码
  result.setPageNum(searchParam.getPageNum());

  //模块
  result.setModule(searchParam.getModule());

  //校区
  result.setCampus(searchParam.getCampus());

  //总页数
  Long totalPage = total / EsContent.PAGE_SIZES;
  result.setTotalPage(total % EsContent.PAGE_SIZES == 0 ? totalPage : totalPage + 1);

  return result;
}
```













