---
layout: post
title: "Springboot整合ElasticSearch采用RestHighClient的方式"
date: 2020-8-31
categories: 后端 运维
tags: SpringBoot Elasticsearch
--- 


### 添加依赖

首先需要创建SpringBoot项目,并在项目的pom.xml文件中添加如下依赖：

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>1.8</java.version>
    <skipTests>true</skipTests>
</properties>


<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.0.4.RELEASE</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- Java High Level REST Client -->
    <dependency>
        <groupId>org.elasticsearch.client</groupId>
        <artifactId>elasticsearch-rest-high-level-client</artifactId>
        <version>6.6.0</version>
    </dependency>
    <dependency>
        <groupId>org.elasticsearch</groupId>
        <artifactId>elasticsearch</artifactId>
        <version>6.6.0</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.0</version>
    </dependency>
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>fastjson</artifactId>
        <version>1.2.60</version>
    </dependency>
    <dependency>
        <groupId>commons-lang</groupId>
        <artifactId>commons-lang</artifactId>
        <version>2.6</version>
    </dependency>
</dependencies>
```
### 添加配置

在resources目录下创建application.yml配置文件，配置如下：

```yaml
spring:
  application:
    name: springboot-es

elasticsearch:
  host: 192.168.192.11
  port: 9200
  schema: http
  index:
    number-of-shards: 3
    number-of-replicas: 1
```

### 具体实现

ElasticsearchRestClient.java代码如下：

```java
@Configuration
public class ElasticsearchRestClient {


    @Value("${elasticsearch.host}")
    private String esHost;

    @Value("${elasticsearch.port}")
    private int esPort;

    @Bean
    public RestClientBuilder restClientBuilder() {
        return RestClient.builder(new HttpHost(esHost, esPort, "http"));
    }


    @Bean
    public RestHighLevelClient highLevelClient(@Autowired RestClientBuilder restClientBuilder) {
        restClientBuilder.setMaxRetryTimeoutMillis(60000);
        return new RestHighLevelClient(restClientBuilder);
    }
}
```

Item.java代码如下:

```java
@Slf4j
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Item {

    // ES主键
    private String _id;

    // 数据库主键
    private Long id;

    //标题
    private String title;

    // 分类
    private String category;

    // 品牌
    private String brand;

    // 价格
    private Double price;

    // 图片地址
    private String images;


    public static XContentBuilder buildMapping (){
        XContentBuilder builder = null;
        try {
            builder = JsonXContent.contentBuilder()
                    .startObject()
                        .startObject("properties")
                            .startObject("id")
                            .field("type", "long")
                            .field("index", "true")
                            .endObject()

                            .startObject("title")
                            .field("type", "text")
                            .field("index", "true")
			    .field("analyzer", "ik_max_word")
                            .endObject()

                            .startObject("category")
                            .field("type", "keyword")
                            .field("index", "true")
                            .endObject()

                            .startObject("brand")
                            .field("type", "keyword")
                            .field("index", "true")
                            .endObject()

                            .startObject("price")
                            .field("type", "double")
                            .field("index", "true")
                            .endObject()

                            .startObject("images")
                            .field("type", "keyword")
                            .field("index", "true")
                            .endObject()
                        .endObject()
                    .endObject();
        } catch (IOException e) {
            log.error(e.getMessage(), e);
        }
        return builder;
    }
}
```


> ik 带有两个分词器：
> ik_max_word：会将文本做最细粒度的拆分；尽可能多的拆分出词语
> ik_smart：会做最粗粒度的拆分；已被分出的词语将不会再次被其它词语占有


EsCurdService.java代码如下：

```java
public interface EsCurdService {

    /**
     * 创建索引
     * @param indexName 索引名称
     * @return
     */
    public void createIndex(String indexName);

    /**
     * 创建索引
     * @param indexName 索引名称
     * @param type      数据对象类型，可以把它设置成对象类的类名
     * @param builder
     */
    public void createIndex(String indexName, String type, XContentBuilder builder);


    /**
     * 判断索引是否存在
     * @param indexName
     * @return
     */
    public boolean existIndex(String indexName);


    /**
     * 删除索引
     * @param indexName 索引名称
     */
    public void deleteIndex(String indexName);


    /**
     * 新增数据
     * @param indexName 索引名称
     * @param type      数据对象类型，可以把它设置成对象类的类名
     * @param object    数据对象
     * @return
     */
    public RestStatus insert(String indexName, String type, Object object);

    /**
     *
     * 修改数据
     * @param indexName  索引名称
     * @param type       数据对象类型，可以把它设置成对象类的类名
     * @param id        主键
     * @param object    数据对象
     * @return
     */
    public RestStatus update(String indexName, String type, String id, Object object);


    /**
     * 删除数据
     * @param indexName 索引名称
     * @param type      数据对象类型，可以把它设置成对象类的类名
     * @param id        主键 RsBwNHQBWruqdhIY3dgZ
     * @return
     */
    public RestStatus delete(String indexName, String type, String id);


    /**
     * 数据查询
     * @param indexName
     * @param type
     * @param builder
     * @return
     */
    public SearchResponse search(String indexName, String type, SearchSourceBuilder builder);
}

```

EsCurdServiceImpl.java代码如下：

```java
@Slf4j
@Service
public class EsCurdServiceImpl implements EsCurdService {

    @Value("${elasticsearch.index.number-of-shards:3}")
    private int number_of_shards;

    @Value("${elasticsearch.index.number-of-replicas:2}")
    private int number_of_replicas;

    @Autowired
    private RestHighLevelClient highLevelClient;

    @Override
    public void createIndex(String indexName) {
        // 判断是否存在索引
        if(existIndex(indexName)) {
            return;
        }

        // 创建索引
        CreateIndexRequest createIndexRequest = new CreateIndexRequest();
        createIndexRequest.index(indexName).settings(Settings.builder()
                // 分片数
                .put("index.number_of_shards", number_of_shards)
                // 备份数
                .put("index.number_of_replicas", number_of_replicas));
        CreateIndexResponse response = null;
        try {
            response = highLevelClient.indices().create(createIndexRequest, RequestOptions.DEFAULT);
        } catch (IOException e) {
            log.info(e.getMessage(), e);
        }
    }

    @Override
    public void createIndex(String indexName, String type, XContentBuilder builder) {
        // 判断是否存在索引
        if(existIndex(indexName)) {
            return;
        }

        // 创建索引
        CreateIndexRequest createIndexRequest = new CreateIndexRequest();
        createIndexRequest.index(indexName).settings(Settings.builder()
                // 分片数
                .put("index.number_of_shards", number_of_shards)
                // 备份数
                .put("index.number_of_replicas", number_of_replicas))
                // 创建映射
                .mapping(type, builder);
        CreateIndexResponse response = null;
        try {
            response = highLevelClient.indices().create(createIndexRequest, RequestOptions.DEFAULT);
        } catch (IOException e) {
            log.info(e.getMessage(), e);
        }
    }

    @Override
    public boolean existIndex(String indexName) {
        // 判断索引是否存在
        boolean isExist = false;
        GetIndexRequest getIndexRequest = new GetIndexRequest();
        getIndexRequest.indices(indexName);
        try {
            isExist = highLevelClient.indices().exists(getIndexRequest, RequestOptions.DEFAULT);
        } catch (IOException e) {
            log.info(e.getMessage(), e);
        }
        return isExist;
    }



    @Override
    public void deleteIndex(String indexName) {
        DeleteIndexRequest deleteIndexRequest = new DeleteIndexRequest();
        deleteIndexRequest.indices(indexName);
        try {
            highLevelClient.indices().delete(deleteIndexRequest, RequestOptions.DEFAULT);
        } catch (IOException e) {
            log.info(e.getMessage(), e);
        }
    }


    @Override
    public RestStatus insert(String indexName, String type, Object object) {
        IndexRequest indexRequest = new IndexRequest();
        // source方法里面需要填写对应数据的类型，默认数据类型为json
        indexRequest.index(indexName).type(type).source(JSON.toJSONString(object), XContentType.JSON);
        IndexResponse response = null;
        try {
            response = highLevelClient.index(indexRequest, RequestOptions.DEFAULT);
            return response.status();
        } catch (IOException e) {
            log.info(e.getMessage(), e);
        }

        return null;
    }


    @Override
    public RestStatus update(String indexName, String type, String id, Object object) {
        UpdateRequest request = new UpdateRequest();
        request.index(indexName).type(type).id(id).doc(JSON.toJSONString(object), XContentType.JSON);
        UpdateResponse response = null;
        try {
            response = highLevelClient.update(request, RequestOptions.DEFAULT);
            return response.status();
        } catch (IOException e) {
            log.info(e.getMessage(), e);
        }
        return null;
    }


    @Override
    public RestStatus delete(String indexName, String type, String id) {
        DeleteRequest request = new DeleteRequest();
        request.index(indexName).type(type).id(id);
        DeleteResponse response = null;
        try {
            response = highLevelClient.delete(request, RequestOptions.DEFAULT);
            return response.status();
        } catch (IOException e) {
            log.info(e.getMessage(), e);
        }
        return null;
    }



    @Override
    public SearchResponse search(String indexName, String type, SearchSourceBuilder sourceBuilder) {
        SearchRequest searchRequest = new SearchRequest();
        searchRequest.indices(indexName).types(type).source(sourceBuilder);
        SearchResponse response = null;
        try {
            response = highLevelClient.search(searchRequest, RequestOptions.DEFAULT);
            return response;
        } catch (IOException e) {
            log.info(e.getMessage(), e);
        }
        return null;
    }
}
```

IndexController.java代码如下：

```java
@Slf4j
@RestController
@RequestMapping("/es")
public class IndexController {

    @Autowired
    private EsCurdService esCurdService;

    // es索引(相当于数据库)
    public static final String ES_INDEX_NAME = "es-index";

    // es文档类型(相当于库表)
    public static final String ES_DOC_TYPE = "item";

    @GetMapping("/add")
    public void create(){
        Item item = Item.builder()
                .id(1L)
                .title("小米手机7")
                .category("手机")
                .brand("小米")
                .images("http://image.leyou.com/13123.jpg")
                .build();
        XContentBuilder builder = Item.buildMapping();
        esCurdService.createIndex(ES_INDEX_NAME, ES_DOC_TYPE, builder);
        esCurdService.insert(ES_INDEX_NAME, ES_DOC_TYPE, item);
    }


    @GetMapping("/update")
    public void update(){
        Item item = Item.builder()
                .id(1L)
                .title("小米手机8")
                .category("手机")
                .brand("小米")
                .images("http://image.leyou.com/13123.jpg")
                .build();

        String _id = "RsBwNHQBWruqdhIY3dgZ";

        esCurdService.update(ES_INDEX_NAME, ES_DOC_TYPE, _id, item);
    }


    @GetMapping("/list")
    public List<Item> list(){
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        // 分页参数
        sourceBuilder.from(0);
        sourceBuilder.size(10);

        BoolQueryBuilder boolBuilder = QueryBuilders.boolQuery();
        // 关键词查找匹配，这个不是精确查找，他会对字段中的内容进行切词匹配
        MatchQueryBuilder matchQueryBuilder = QueryBuilders.matchQuery("title", "小米");
        boolBuilder.must(matchQueryBuilder);
        // 设置需要返回的字段
        sourceBuilder.fetchSource(new String[]{"id", "title", "category"}, new String[]{}).query(boolBuilder);

        // 高亮
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        highlightBuilder.preTags("<span style=\"color:red\">"); //前置元素
        highlightBuilder.postTags("</span>");  //后置元素
        highlightBuilder.fields().add(new HighlightBuilder.Field("title"));  //高亮查询字段
        highlightBuilder.requireFieldMatch(false);     //如果要多个字段高亮,这项要为false
        sourceBuilder.highlighter(highlightBuilder);

        SearchResponse response =  esCurdService.search(ES_INDEX_NAME, ES_DOC_TYPE, sourceBuilder);


        ObjectMapper objectMapper = new ObjectMapper();
        List<Item> result = new ArrayList<>();

        response.getHits().iterator().forEachRemaining(hit -> {
            Map<String, HighlightField> highlightFields  = hit.getHighlightFields();
            HighlightField content = highlightFields.get("title");
            //fragments就是高亮显示的片段
            Text[] fragments = content.fragments();
            StringBuffer sb = new StringBuffer();
            //拼接查询出来的高亮片段
            for (Text text:fragments){
                sb.append(text);
            }

            //将查询结果转化为对象--因为我们要进行替换，所以要转化成对象，然后直接将高亮片段替换原有对象内容
            try {
                Item item = objectMapper.readValue(hit.getSourceAsString() , Item.class);
                // 设置ES中主键
                item.set_id(hit.getId());

                //开始进行高亮替换
                if(StringUtils.isNotEmpty(sb.toString())) {
                    item.setTitle(sb.toString());
                }
                result.add(item);
            } catch (IOException e) {
                log.error(e.getMessage(), e);
            }
        });

        return result;
    }
}
```

本文代码git地址 [https://github.com/chaojunma/springboot-es.git](https://github.com/chaojunma/springboot-es.git)