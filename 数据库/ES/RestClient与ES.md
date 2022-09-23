# RestClient与ES
## 依赖

新增：

    <dependency>
         <groupId>org.elasticsearch.client</groupId>
         <artifactId>elasticsearch-rest-high-level-client</artifactId>
         <version>7.12.1</version>
    </dependency>

添加：因为有两个子jar版本不一样，所以还要单独指定一下
    
    <elasticsearch.version>7.12.1</elasticsearch.version>
    
![](images/86330768.png)

## 快速入门match_all
    
> 我们以match_all查询为例

### 发送查询请求
![](images/78c41fc2.png)

代码解读：

- 第一步，创建SearchRequest对象，指定索引库名

- 第二步，利用request.source()构建DSL，DSL中可以包含查询、分页、排序、高亮等
    - query()：代表查询条件，利用QueryBuilders.matchAllQuery()构建一个match_all查询的DSL
- 第三步，利用client.search()发送请求，得到响应

> 两个关键的API

- **request.source()**，其中包含了查询、排序、分页、高亮等所有功能：
  
- **QueryBuilders**，其中包含match、term、function_score、bool等各种查询：

![](images/b937ee5f.png)

![](images/049bb358.png)

### 解析响应结果

![](images/eb2e2c2c.png)

响应结果的解析：
- elasticsearch返回的结果是一个JSON字符串，结构包含：

    - hits：命中的结果
    - total：总条数，其中的value是具体的总条数值
    - max_score：所有结果中得分最高的文档的相关性算分
    - hits：搜索结果的文档数组，其中的每个文档都是一个json对象
    - _source：文档中的原始数据，也是json对象
    
因此，我们解析响应结果，就是逐层解析JSON字符串，流程如下：

- SearchHits：通过response.getHits()获取，就是JSON中的最外层的hits，代表命中的结果
    - SearchHits#getTotalHits().value：获取总条数信息
    - SearchHits#getHits()：获取SearchHit数组，也就是文档数组
        - SearchHit#getSourceAsString()：获取文档结果中的_source，也就是原始的json文档数据
        
### 完整代码

    @SpringBootTest
    public class HotelDataTest {
        private RestHighLevelClient client;
        @Autowired
        private IHotelService hotelService;
    
        /**
         * 批量添加数据
         */
        @Test
        void matchAllTest() throws IOException {
            // 1.准备Request
            SearchRequest request = new SearchRequest("hotel");
            // 2.准备DSL
            MatchAllQueryBuilder matchAllQueryBuilder = QueryBuilders.matchAllQuery();
            request.source().query(matchAllQueryBuilder);
            request.source().from(10);
            request.source().size(20);
            // 3.发送请求
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
            // 4.解析响应
    
            handleResponse(response);
    
        }
    
        /**
         * 解析响应结果
         *
         * @param response
         */
        private void handleResponse(SearchResponse response) {
            SearchHits searchHits = response.getHits();
            long total = searchHits.getTotalHits().value;
            if (total > 0) {
                System.out.println("一共得到" + total + "条数据");
                SearchHit[] hits = searchHits.getHits();
                for (SearchHit hit : hits) {
                    String dataJson = hit.getSourceAsString();
                    HotelDoc hotelDoc = JSONUtil.toBean(dataJson, HotelDoc.class);
                    System.out.println("hotelDoc==" + hotelDoc);
                }
            }
    
        }
    
        @BeforeEach
        void setUp() {
            client = new RestHighLevelClient(RestClient.builder(HttpHost.create("http://192.168.171.132:9200")));
        }
    
        @AfterEach
        void tearDown() throws IOException {
            client.close();
        }
    }

## match查询

> 全文检索的match和multi_match查询与match_all的API基本一致。差别是查询条件，也就是query的部分。

         @Test
        void matchTest() throws IOException {
            // 1.准备Request
            SearchRequest request = new SearchRequest("hotel");
            // 2.准备DSL
            MatchQueryBuilder builder = QueryBuilders.matchQuery("all", "如家");
            request.source().query(builder);
            request.source().from(10);
            request.source().size(20);
            // 3.发送请求
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
            // 4.解析响应
    
            handleResponse(response);
    
        }   
        
        

## wildcard模糊查询

     @Test
        void wildcardTest() throws IOException {
            // 1.准备Request
            SearchRequest request = new SearchRequest("hotel");
            // 2.准备DSL
            WildcardQueryBuilder builder = QueryBuilders.wildcardQuery("all", "北*");
            request.source().query(builder);
            request.source().from(10);
            request.source().size(20);
            // 3.发送请求
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
            // 4.解析响应
    
            handleResponse(response);
    
        }


## 精确查询
精确查询主要是两者：

- term：词条精确匹配
- range：范围查询

与之前的查询相比，差异同样在查询条件，其它都一样。
![](images/9d704fea.png)


     @Test
        void rangeTest() throws IOException {
            // 1.准备Request
            SearchRequest request = new SearchRequest("hotel");
            // 2.准备DSL
            RangeQueryBuilder builder = QueryBuilders.rangeQuery("price").gte(500).lte(1000);
            request.source().query(builder);
            request.source().from(10);
            request.source().size(20);
            // 3.发送请求
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
            // 4.解析响应
    
            handleResponse(response);
    
        }
        

## 布尔查询

布尔查询是用must、must_not、filter等方式组合其它查询，代码示例如下： 

![](images/bf66c939.png)

可以看到，API与其它查询的差别同样是在查询条件的构建，QueryBuilders，结果解析等其他代码完全不变。  

    
    @Test
        void boolTest() throws IOException {
            // 1.准备Request
            SearchRequest request = new SearchRequest("hotel");
            // 2.准备DSL
            BoolQueryBuilder builder = QueryBuilders.boolQuery();
            //范围查询
    //        builder.filter(QueryBuilders.rangeQuery("price").gte(500).lte(1000));
    
            builder.must(QueryBuilders.termQuery("name","如家"));
            request.source().query(builder);
            request.source().from(10);
            request.source().size(20);
            // 3.发送请求
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
            // 4.解析响应
    
            handleResponse(response);
    
        }
        
        

## 排序、分页

     @Test
        void testPageAndSort() throws IOException {
            // 页码，每页大小
            int page = 1, size = 5;
            // 1.准备Request
            SearchRequest request = new SearchRequest("hotel");
            // 2.准备DSL
            MatchAllQueryBuilder builder = QueryBuilders.matchAllQuery();
    
            request.source().query(builder);
            request.source().from(page - 1);
            request.source().size(size);
            // 3.发送请求
            SearchResponse response = client.search(request, RequestOptions.DEFAULT);
            // 4.解析响应
            handleResponse(response);
        }
        

## 高亮
### 发送请求
    
             @Test
            void testHighlight() throws IOException {
                // 1.准备Request
                SearchRequest request = new SearchRequest("hotel");
                // 2.准备DSL
                MatchQueryBuilder builder = QueryBuilders.matchQuery("all", "如家");
                request.source().query(builder);
                // requireFieldMatch(false)表示不要求整个字段相同，默认true
                request.source().highlighter(
                        new HighlightBuilder().field("name").requireFieldMatch(false)
                );
        
                // 3.发送请求
                SearchResponse response = client.search(request, RequestOptions.DEFAULT);
                // 4.解析响应
                handleResponse(response);
            }
  
            
### 返回结构

    private void handleResponse(SearchResponse response) {
            SearchHits searchHits = response.getHits();
            long total = searchHits.getTotalHits().value;
            if (total > 0) {
                System.out.println("一共得到" + total + "条数据");
                SearchHit[] hits = searchHits.getHits();
                for (SearchHit hit : hits) {
                    String dataJson = hit.getSourceAsString();
                    HotelDoc hotelDoc = JSONUtil.toBean(dataJson, HotelDoc.class);
                    //高亮结果分析
                    Map<String, HighlightField> fields = hit.getHighlightFields();
                    if(!fields.isEmpty()){
                        HighlightField highlightField = fields.get("name");
                        if(highlightField!=null){
                            String name = highlightField.getFragments()[0].toString();
                            hotelDoc.setName(name);
                        }
                    }
    
                    System.out.println("hotelDoc==" + hotelDoc);
                }
            }
        }  


