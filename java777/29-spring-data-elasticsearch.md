# 29 · Spring Data Elasticsearch

## 1. 核心概念

| ES 概念 | 类比 MySQL |
|---|---|
| Index（索引） | 表 |
| Document（文档） | 行 |
| Field | 列 |
| Mapping | 表结构（字段类型） |
| DSL（JSON 查询） | SQL |
| Shard（分片） | 分表 |
| Replica | 备库 |

- **倒排索引**：term → 文档列表，全文检索快的原因。
- **分词**：`text` 会分词（用于搜索），`keyword` 不分词（用于精确匹配/聚合）。
- **近实时**：默认 1s refresh，写入后约 1s 可搜到。

## 2. 依赖与配置（ES 8.x）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
```

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: xxx
    connection-timeout: 3s
    socket-timeout: 10s
```

> 版本必须匹配：Boot 3.2 对应 ES 8.11+。生产推荐用原生 `ElasticsearchClient`（Java API Client）而非 Spring Data（后者抽象较重、跟进慢）。

## 3. 实体映射

```java
@Document(indexName = "product")
@Setting(shards = 3, replicas = 1)
@Data
public class ProductDoc {
    @Id private String id;

    @Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart")
    private String name;

    @Field(type = FieldType.Keyword)
    private String category;

    @Field(type = FieldType.Double)
    private BigDecimal price;

    @Field(type = FieldType.Integer)
    private Integer stock;

    @Field(type = FieldType.Date, format = DateFormat.date_hour_minute_second)
    private LocalDateTime createTime;

    @Field(type = FieldType.Text, analyzer = "ik_max_word")
    private String description;

    @CompletionField(maxInputLength = 100)     // 自动补全
    private Completion suggest;

    @GeoPointField                              // 地理位置
    private GeoPoint location;
}
```

> 中文分词需安装 **IK 分词器**插件（`ik_max_word` 细粒度索引、`ik_smart` 粗粒度查询）。

## 4. Repository 方式

```java
public interface ProductRepository extends ElasticsearchRepository<ProductDoc, String> {
    List<ProductDoc> findByName(String name);                 // 精确/匹配
    List<ProductDoc> findByCategoryAndPriceBetween(String c, BigDecimal a, BigDecimal b);
    Page<ProductDoc> findByNameLike(String kw, Pageable p);

    @Highlight(fields = @HighlightField(name = "name"))
    @Query("{\"match\":{\"name\":\"?0\"}}")
    SearchHits<ProductDoc> searchHighlight(String kw);
}
```

## 5. ElasticsearchTemplate / ElasticsearchClient（推荐）

```java
@Service
@RequiredArgsConstructor
public class ProductSearchService {
    private final ElasticsearchClient client;

    public SearchResult<ProductDoc> search(ProductQuery q) {
        // Bool 查询
        BoolQuery.Builder bool = new BoolQuery.Builder();
        if (StringUtils.hasText(q.getKeyword())) {
            bool.must(MultiMatchQuery.of(m -> m
                    .fields("name^3", "description")       // name 权重 3 倍
                    .query(q.getKeyword())
                    .operator(Operator.And))._toQuery());
        }
        if (q.getCategory() != null)
            bool.filter(TermQuery.of(t -> t.field("category").value(q.getCategory()))._toQuery());
        if (q.getMinPrice() != null)
            bool.filter(RangeQuery.of(r -> r.field("price").gte(JsonData.of(q.getMinPrice())))._toQuery());

        // 分页 + 排序 + 高亮
        SearchRequest req = SearchRequest.of(s -> s
            .index("product")
            .query(Query.of(qb -> qb.bool(bool.build())))
            .from((q.getPageNo() - 1) * q.getPageSize())
            .size(q.getPageSize())
            .sort(SortOptions.of(so -> so.field(f -> f.field("createTime").order(SortOrder.Desc))))
            .highlight(h -> h.fields("name", hf -> hf
                     .preTags("<em>").postTags("</em>")))
            .source(src -> src.filter(sf -> sf.excludes("description")))
        );

        SearchResponse<ProductDoc> resp = client.search(req, ProductDoc.class);
        return convert(resp);      // hits.total + hits + highlight 片段
    }
}
```

## 6. 常用查询 DSL

| 场景 | DSL |
|---|---|
| 精确 | `term`（keyword 字段） |
| 分词匹配 | `match` / `multi_match` |
| 短语 | `match_phrase` |
| 范围 | `range`（gte/lte） |
| 组合 | `bool`（must/should/filter/must_not） |
| 聚合 | `terms` / `date_histogram` / `avg` / `cardinality` |
| 前缀/通配 | `prefix` / `wildcard`（避免前导 `*`） |
| 模糊 | `fuzzy` |
| 嵌套 | `nested` |
| 打分控制 | `function_score`、`boost` |

> **`filter` vs `must`**：filter 不参与打分、可缓存，性能好；must 参与评分。

## 7. 聚合示例

```java
SearchResponse<Void> r = client.search(s -> s.index("product").size(0)
    .aggregations("by_cat", a -> a.terms(t -> t.field("category").size(20))
        .aggregations("avg_price", a2 -> a2.avg(v -> v.field("price")))), Void.class);
```

## 8. 深分页问题

| 方案 | 说明 |
|---|---|
| `from + size` | 默认上限 `index.max_result_window=10000`，深分页内存爆炸 |
| `scroll` | 快照式，适合**批量导出**（实时性差） |
| `search_after` | **推荐**实时深分页，用上一页排序值做游标 |
| `PIT` | Point in Time，配合 search_after 保证一致性 |

```java
// search_after
.sort(...) ... .searchAfter(List.of(FieldValue.of(lastTime), FieldValue.of(lastId)))
```

## 9. 数据同步（DB → ES）

| 方案 | 说明 | 适用 |
|---|---|---|
| 双写 | 代码里同时写 DB 与 ES | 简单，一致性差 |
| 定时全量/增量扫表 | 按 `update_time` 增量 | 延迟高 |
| MQ 异步 | 变更发 MQ，消费写 ES | 常用，需保证顺序 |
| Canal/Debezium 订阅 binlog | 解耦、可靠 | **推荐** |
| ES 作为唯一存储 | 无 DB | 适合搜索型业务 |

一致性保障：版本号乐观锁（ES 的 `_seq_no` / `if_primary_term`）或按主键幂等覆盖写。

## 10. 写入优化

- **批量**：`BulkRequest`（1000~5000 条/批，5~15MB）。
- 写入期间可临时 `refresh_interval: -1`、`replicas: 0`，完成后恢复。
- 避免大文档；避免频繁更新（ES 是段合并，更新 = 删+插）。
- 索引按时间滚动（`product-2024-01`）+ 别名。

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 搜不到刚写的数据 | 近实时 1s | 写后 `?refresh=true` 或等待 |
| `text` 字段聚合报错 | Fielddata disabled | 用 `keyword` 子字段 |
| 中文搜不到 | 未装 IK | 安装插件并指定 analyzer |
| 深分页报错 | `Result window is too large` | `search_after` |
| Mapping 冲突 | 字段类型不一致 | 先建 mapping 再写数据 |
| 字段类型改不了 | 已存在 mapping | 新建索引 + reindex + 别名切换 |
| 数据不一致 | 同步失败 | MQ 重试 + 定时对账 |
| 分片过多 | 性能下降 | 单分片 20~50GB 为宜 |

## 12. 面试速答

- **ES 为什么快**：倒排索引 + 分词 + 段（segment）不可变 + 缓存（filter cache）+ 分布式并行。
- **text 与 keyword**：text 分词用于搜索，不能聚合；keyword 不分词，用于精确匹配、排序、聚合。
- **深分页怎么做**：`search_after`（实时）或 `scroll`（离线导出），避免大 `from`。
- **DB 与 ES 一致性**：binlog 订阅（Canal）+ MQ 重试 + 幂等写入 + 定时对账。
