ES里的 Index 可以看做一个库，而 Types 相当于表， Documents 则相当于表的行。这里Types 的概念已经被逐渐弱化， Elasticsearch 6.X 中，一个 index 下已经只能包含一个type Elasticsearch 7.X 中 , Type 的概念已经被删除了。

# HTTP操作

## 1、索引操作

### 1、创建索引

对比关系型数据库，创建索引就等同于创建数据库

put请求  http://127.0.0.1:9200/start  创建索引start

### 2、查看索引

向 ES 服务器发 GET 请求 http://127.0.0.1:9200/_cat/indices?v

这里请求路径中的 _cat 表示查看的意思， indices 表示索引，所以整体含义就是查看当前 ES服务器中的所有索引，就好像 MySQL 中的 show tables 的感觉，服务器响应结果如下

### 3、查看单个索引

在Postman 中，向 ES 服务器发 GET 请求 http://127.0.0.1:9200/start

### 4、删除索引

在Postman 中，向 ES 服务器发 DELETE 请求 http://127.0.0.1:9200/start

## 2、文档操作

### 1、创建文档

索引已经创建好了，接下来我们来创建文档，并添加数据。这里的文档可以类比为关系型数据库中的表数据，添加的数据格式为 JSON 格式

在Postman 中，向 ES 服务器发 POST 请求 http://127.0.0.1:9200/start/doc

此处发送请求的方式必须为POST ，不能是 PUT ，否则会发生错误

参数例如

{
    "title":"小米手机",
    "category":"小米",
    "price":3999.00
}

上面的数据创建后，由于没有指定数据唯一性标识（ID ），默认情况下 ES 服务器会随机生成一个 。

如果想要自定义唯一性标识，需要在创建时指定http://127.0.0.1:9200/start/doc/1 or http://127.0.0.1:9200/start/_doc/1

### 2、查看文档

查看文档时，需要指明文档的唯一性标识，类似于MySQL 中数据的主键查询

在Postman 中，向 ES 服务器发 GET 请求 http://127.0.0.1:9200/start/_doc/1

### 3、修改文档

和新增文档一样，输入相同的URL 地址请求，如果请求体变化，会将原有的数据内容覆盖 在Postman 中，向 ES 服 务器发 POST 请求 http://127.0.0.1:9200/start/_doc/1

### 4、修改字段

修改数据时，也可以只修改某一给条数据的局部信息

在Postman 中，向 ES 服务器发 POST 请求 http://127.0.0.1:9200/start/_update/1

{
    "doc":{
        "price":6999.00
    }
}

### 5、删除文档

删除一个文档不会立即从磁盘上移除，它只是被标记成已删除（逻辑删除）。

在Postman 中，向 ES 服务器发 DELETE 请求 http://127.0.0.1:9200/start/_doc/1

### 6、条件删除文档

一般删除数据都是根据文档的唯一性标识进行删除，实际操作时，也可以根据条件对多条数据进行删除

向ES 服务器发 POST 请求 http://127.0.0.1:9200/start/_delete_by_query

请求体

{
    "query":{
        "match":{
            "price": 3999.00
        }
    }
}

## 3、映射操作

有了索引库，等于有了数据库中的database 。

接下来就需要建索引库(index)中的映射了，类似于数据库 (database)中的表结构 (table)。创建数据库表需要设置字段名称，类型，长度，约束等；索引库也一样，需要知道这个类型下有哪些字段，每个字段有哪些约束信息，这就叫做映射 (mapping)。

### 1、创建映射

在Postman 中，向 ES 服务器发 PUT 请求http://127.0.0.1:9200/user/_mapping

请求内容为：

{
    "properties":{
        "name":{
            "type": "text",
            "index": true
        },
        "sex":{
            "type": "keyword",
            "index": true
        },
        "phone":{
            "type": "keyword",
            "index": false
        }
    }
}

```java
映射数据说明

字段名：任意填写，下面指定许多属性，例如： title 、 subtitle 、 images 、 price

-  type ：类型 Elasticsearch 中支持的数据类型非常丰富，说几个关键的：

        基本数据类型：long 、 integer 、 short 、 byte 、 double 、 float 、 half_float

            浮点数的高精度类型：scaled_float

        text：可分词

        keyword：不可分词，数据会作为完整字段进行匹配

        String 类型，又分两种：

        Numerical ：数值类型，分两类

        Date ：日期类型

        Array ：数组类型

        Object ：对象

-  index ：是否索引，默认为 true ，也就是说你不进行任何配置，所有字段都会被索引。

        true：字段会被索引，则可以用来进行搜索

        false：字段不会被索引，不能用来搜索

-  store ：是否将数据进行独立存储，默认为 false

        原始的文本会存储在_source 里面，默认情况下其他提取出来的字段都不是独立存储的，是从 _source 里面提取出来的。当然你也可以独立的存储某个字段，只要设置"store": true 即可，获取独立存储的字段要比从 _source 中解析快得多，但是也会占用更多的空间，所以要根据实际业务需求来设置。

-  analyzer ：分词器，这里的 ik_max_word 即使用 ik 分词器

```

### 2、查看映射

在Postman 中，向 ES 服务器发 GET 请求http://127.0.0.1:9200/user/_mapping

### 3、索引映射关联

在Postman 中，向 ES 服务器发 PUT 请求 http://127.0.0.1:9200/user1

# Java API

## 1、环境准备

1、创建项目

2、导入依赖

```xml
<dependency>
   <groupId>org.elasticsearch</groupId>
   <artifactId>elasticsearch</artifactId>
   <version>7.8.0</version>
</dependency>
<!--es的客户端-->
<dependency>
   <groupId>org.elasticsearch.client</groupId>
   <artifactId>elasticsearch-rest-high-level-client</artifactId>
   <version>7.8.0</version>
</dependency>

```

3、创建测试

```java
public class ElasticSearchTest {
 
    public static void main(String[] args) throws IOException {
        //9200 端口为 Elastic s earch 的 Web 通信端口 localhost 为启动 ES 服务的主机名
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost",9200)));
        client.close();
    }
}
```

没有任何输出或报错信息即是成功

## 2、索引操作

### 1、创建索引

ES服务器正常启动后，可以通过 Java API 客户端对象对 ES 索引进行操作

```java
public class ElasticsearchDocCreate {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost",9200)));
        //创建索引
        CreateIndexRequest request = new CreateIndexRequest("user");
        CreateIndexResponse response = client.indices().create(request, RequestOptions.DEFAULT);
        //创建索引的响应状态
        boolean acknowledged = response.isAcknowledged();
        System.out.println("响应状态为：" + acknowledged);
        client.close();
    }
}
```

### 2、查看索引

```java
public class ElasticsearchDocSearch {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost",9200)));
        //查询索引
        GetIndexRequest request = new GetIndexRequest("user");
        GetIndexResponse response = client.indices().get(request, RequestOptions.DEFAULT);
        //查询索引的响应状态
        System.out.println(response);
        System.out.println(response.getSettings());
        System.out.println(response.getAliases());
        System.out.println(response.getMappings());
        client.close();
    }
}
```

### 3、删除索引

```java
public class ElasticsearchDocDelete {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost",9200)));
        //删除索引
        DeleteIndexRequest request = new DeleteIndexRequest("user");
        AcknowledgedResponse response = client.indices().delete(request, RequestOptions.DEFAULT);
        //删除索引的响应状态
        System.out.println("删除状态为：" + response.isAcknowledged());
        client.close();
    }
}
```

## 3、文档操作

创建数据模型

```java
public class User {
 
    private String name;
 
    private Integer age;
 
    private String sex;
 
    public User() {
    }
 
    public User(String name, Integer age, String sex) {
        this.name = name;
        this.age = age;
        this.sex = sex;
    }
 
    public String getName() {
        return name;
    }
 
    public void setName(String name) {
        this.name = name;
    }
 
    public Integer getAge() {
        return age;
    }
 
    public void setAge(Integer age) {
        this.age = age;
    }
 
    public String getSex() {
        return sex;
    }
 
    public void setSex(String sex) {
        this.sex = sex;
    }
 
    @Override
    public String toString() {
        return new StringJoiner(", ", User.class.getSimpleName() + "[", "]")
                .add("name='" + name + "'")
                .add("age=" + age)
                .add("sex='" + sex + "'")
                .toString();
    }
}
```

### 1、创建数据，添加到文档

```java
public class ElasticsearchDocInsert {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost",9200)));
 
        IndexRequest indexRequest = new IndexRequest();
        indexRequest.index("user").id("1001");
        //创建数据对象
        User user = new User("xiaobear",18,"boy");
        //数据对象转为JSON
        ObjectMapper mapper = new ObjectMapper();
        String userJson = mapper.writeValueAsString(user);
        indexRequest.source(userJson, XContentType.JSON);
        //获取响应对象
        IndexResponse response = client.index(indexRequest, RequestOptions.DEFAULT);
        System.out.println("_index：" + response.getIndex());
        System.out.println("_id：" + response.getId());
        System.out.println("_result：" + response.getResult());
        client.close();
    }
}
```

### 2、修改文档

```java
public class ElasticsearchDocUpdate {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost",9200)));
        //修改文档
        UpdateRequest request = new UpdateRequest();
        request.index("user").id("1001");
        // 设置请求体，对数据进行修改
        request.doc(XContentType.JSON,"sex","girl");
        UpdateResponse response = client.update(request, RequestOptions.DEFAULT);
        System.out.println("_index：" + response.getIndex());
        System.out.println("_id：" + response.getId());
        System.out.println("_result：" + response.getResult());
        client.close();
    }
}
```

### 3、查询文档

```java
public class ElasticsearchDocGet {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost",9200)));
        //创建请求对象
        GetRequest request = new GetRequest().index("user").id("1001");
        //创建响应对象
        GetResponse response = client.get(request, RequestOptions.DEFAULT);
        // 打印结果信息
        System.out.println("_index:" + response.getIndex());
        System.out.println("_type:" + response.getType());
        System.out.println("_id:" + response.getId());
        System.out.println("source:" + response.getSourceAsString());
        client.close();
    }
}
```

### 4、删除文档

```java
public class ElasticsearchDoc_Delete {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(
                RestClient.builder(new HttpHost("localhost",9200)));
        //创建请求对象
        DeleteRequest request = new DeleteRequest().index("user").id("1");
        //创建响应对象
        DeleteResponse response = client.delete(request, RequestOptions.DEFAULT);
        //打印信息
        System.out.println(response.toString());
        client.close();
    }
}
```

### 5、批量操作

#### 1、批量新增

```
public class ElasticSearchBatchInsert {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建批量新增请求对象
        BulkRequest request = new BulkRequest();
        request.add(new IndexRequest().index("user").id("1004").source(XContentType.JSON,"name","xiaohuahua"));
        request.add(new IndexRequest().index("user").id("1005").source(XContentType.JSON,"name","zhangsan"));
        request.add(new IndexRequest().index("user").id("1006").source(XContentType.JSON,"name","lisi"));
        //创建响应对象
        BulkResponse response = client.bulk(request, RequestOptions.DEFAULT);
        System.out.println("took：" + response.getTook());
        System.out.println("items：" + response.getItems());
    }
}
```

#### 2、批量删除

```java
public class ElasticSearchBatchDelete {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建批量新增请求对象
        BulkRequest request = new BulkRequest();
        request.add(new DeleteRequest().index("user").id("1001"));
        request.add(new DeleteRequest().index("user").id("1002"));
        request.add(new DeleteRequest().index("user").id("1003"));
        //创建响应对象
        BulkResponse response = client.bulk(request, RequestOptions.DEFAULT);
        System.out.println("took：" + response.getTook());
        Arrays.stream(response.getItems()).forEach(System.out::println);
        System.out.println("items：" + Arrays.toString(response.getItems()));
        System.out.println("status：" + response.status());
        System.out.println("失败消息：" + response.buildFailureMessage());
    }
}
```

## 4、高级查询

### 1、请求体查询

#### 1、查询所有索引数据

```java
public class RequestBodyQuery {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建搜索对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        //构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //查询所有对象
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        request.source(sourceBuilder);
 
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took：" + response.getTook());
        System.out.println("是否超时：" + response.isTimedOut());
        System.out.println("TotalHits：" + hits.getTotalHits());
        System.out.println("MaxScore：" + hits.getMaxScore());
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
```

#### 2、term查询

```java
public class TremQuery {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建搜索对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        //构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //查询所有对象
        sourceBuilder.query(QueryBuilders.termQuery("name","zhangsan"));
        request.source(sourceBuilder);
 
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took：" + response.getTook());
        System.out.println("是否超时：" + response.isTimedOut());
        System.out.println("TotalHits：" + hits.getTotalHits());
        System.out.println("MaxScore：" + hits.getMaxScore());
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
```

#### 3、分页查询

```java
public class PageQuery {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建搜索对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        //构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //查询所有对象
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        //分页查询 当前页其实索引 第一条数据的顺序号 from
        sourceBuilder.from(0);
        //每页显示多少条
        sourceBuilder.size(2);
        request.source(sourceBuilder);
 
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took：" + response.getTook());
        System.out.println("是否超时：" + response.isTimedOut());
        System.out.println("TotalHits：" + hits.getTotalHits());
        System.out.println("MaxScore：" + hits.getMaxScore());
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
```

#### 4、数据排序

```java
public class DataSorting {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建搜索对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        //构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //查询所有对象
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        //数据排序
        sourceBuilder.sort("age", SortOrder.DESC);
 
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took：" + response.getTook());
        System.out.println("是否超时：" + response.isTimedOut());
        System.out.println("TotalHits：" + hits.getTotalHits());
        System.out.println("MaxScore：" + hits.getMaxScore());
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
```

#### 5、过滤字段

```java
public class FilterFiled {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建搜索对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        //构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //查询所有对象
        sourceBuilder.query(QueryBuilders.matchAllQuery());
        //数据排序
        sourceBuilder.sort("age", SortOrder.DESC);
        //查询过滤字段
        String[] excludes = {};
        //过滤掉name属性
        String[] includes = {"age"};
        sourceBuilder.fetchSource(includes,excludes);
 
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took：" + response.getTook());
        System.out.println("是否超时：" + response.isTimedOut());
        System.out.println("TotalHits：" + hits.getTotalHits());
        System.out.println("MaxScore：" + hits.getMaxScore());
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
```

#### 6、Bool查询

```java
public class BoolSearch {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建搜索对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        //构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();
        //必须包含
        boolQuery.must(QueryBuilders.matchQuery("age",18));
        //一定不包含
        boolQuery.mustNot(QueryBuilders.matchQuery("name","lisi"));
        //可能包含
        boolQuery.should(QueryBuilders.matchQuery("name","zhangsan"));
        //查询所有对象
        sourceBuilder.query(boolQuery);
        
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took：" + response.getTook());
        System.out.println("是否超时：" + response.isTimedOut());
        System.out.println("TotalHits：" + hits.getTotalHits());
        System.out.println("MaxScore：" + hits.getMaxScore());
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
```

#### 7、范围查询

```java
public class RangeSearch {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建搜索对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        //构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //范围查询
        RangeQueryBuilder rangeQuery = QueryBuilders.rangeQuery("age");
        //大于等于
        rangeQuery.gte("19");
        //小于等于
        rangeQuery.lte("40");
        //查询所有对象
        sourceBuilder.query(rangeQuery);
 
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took：" + response.getTook());
        System.out.println("是否超时：" + response.isTimedOut());
        System.out.println("TotalHits：" + hits.getTotalHits());
        System.out.println("MaxScore：" + hits.getMaxScore());
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
```

#### 8、模糊查询

```java
public class FuzzySearch {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //创建搜索对象
        SearchRequest request = new SearchRequest();
        request.indices("user");
        //构建查询的请求体
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //模糊查询
        FuzzyQueryBuilder fuzzyQuery = QueryBuilders.fuzzyQuery("name", "zhangsan");
        fuzzyQuery.fuzziness(Fuzziness.ONE);
 
        sourceBuilder.query(fuzzyQuery);
 
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //查询匹配
        SearchHits hits = response.getHits();
        System.out.println("took：" + response.getTook());
        System.out.println("是否超时：" + response.isTimedOut());
        System.out.println("TotalHits：" + hits.getTotalHits());
        System.out.println("MaxScore：" + hits.getMaxScore());
        for (SearchHit hit : hits) {
            System.out.println(hit.getSourceAsString());
        }
    }
}
```

#### 9、高亮查询

```java
public class HighlightQuery {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        //高亮查询
        SearchRequest request = new SearchRequest("user");
        //创建查询请求体构建器
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        //构建查询方式，高亮查询
        TermQueryBuilder termQueryBuilder = QueryBuilders.termQuery("name", "zhangsan");
        //设置查询方式
        sourceBuilder.query(termQueryBuilder);
        //构建高亮字段
        HighlightBuilder highlightBuilder = new HighlightBuilder();
        //设置标签前缀
        highlightBuilder.preTags("<font color='red'");
        //设置标签后缀
        highlightBuilder.postTags("</font>");
        //设置高亮字段
        highlightBuilder.field("name");
        //设置高亮构建对象
        sourceBuilder.highlighter(highlightBuilder);
        //设置请求体
        request.source(sourceBuilder);
        //客户端发送请求，获取响应对象
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        // 打印响应结果
        SearchHits hits = response.getHits();
        System.out.println("took::"+response.getTook());
        System.out.println("time_out::"+response.isTimedOut());
        System.out.println("total::"+hits.getTotalHits());
        System.out.println("max_s core::"+hits.getMaxScore());
        System.out.println("hits::::>>");
        for (SearchHit hit : hits) {
            String sourceAsString = hit.getSourceAsString();
            System.out.println(sourceAsString);
            //打印高亮结果
            Map<String, HighlightField> highlightFields = hit.getHighlightFields();
            System.out.println(highlightFields);
            System.out.println("<<::::");
        }
    }
}
```

### 2、聚合查询

#### 1、最大年龄

```java
public class AggregateQuery {
 
    public static void main(String[] args) throws IOException {
        RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
        SearchRequest request = new SearchRequest().indices("user");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        sourceBuilder.aggregation(AggregationBuilders.max("maxAge").field("age"));
        //设置请求体
        request.source(sourceBuilder);
        SearchResponse response = client.search(request, RequestOptions.DEFAULT);
        //打印响应结果
        SearchHits hits = response.getHits();
        System.out.println("hits = " + hits);
        System.out.println(response);
    }
}
```

#### 2、分组查询

```java
 public class GroupQuery {
 
     public static void main(String[] args) throws IOException {
         RestHighLevelClient client = new RestHighLevelClient(RestClient.builder(new HttpHost("localhost", 9200)));
         //创建搜索对象
         SearchRequest request = new SearchRequest().indices("user");
         SearchSourceBuilder searchSourceBuilder = new SearchSourceBuilder();
         searchSourceBuilder.aggregation(AggregationBuilders.terms("age_groupby").field("age"));
 
         //设置请求体
         request.source(searchSourceBuilder);
         SearchResponse response = client.search(request, RequestOptions.DEFAULT);
         System.out.println(response.getHits());
         System.out.println(response);
 
     }
 }
```
