# 55 · Spring Batch 批处理

## 1. 适用场景

处理**大量数据**的离线任务：数据迁移、对账、报表生成、批量发券、日终结算。

特点：分块处理、事务性、可重启、可监控、失败可跳过/重试。

## 2. 核心概念

```
Job（作业）
 └── Step（步骤）
      ├── ItemReader   读
      ├── ItemProcessor 处理（可选）
      └── ItemWriter   写
          （chunk = 读 N 条 → 处理 → 一次写，N 条一个事务）
```

| 概念 | 说明 |
|---|---|
| `JobRepository` | 存储作业元数据（执行状态、参数），需建表 |
| `JobLauncher` | 启动作业 |
| `ItemReader` / `Processor` / `Writer` | 读写处理 |
| `Chunk` | 提交间隔（默认 10） |
| `JobParameters` | 作业参数（**相同参数只能成功执行一次**，用于幂等） |
| `StepExecution` / `JobExecution` | 执行上下文 |
| `ExecutionContext` | 步骤间传递数据、重启点 |

## 3. 依赖与配置

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-batch</artifactId>
</dependency>
<dependency><groupId>com.mysql</groupId><artifactId>mysql-connector-j</artifactId></dependency>
```

```yaml
spring:
  batch:
    jdbc:
      initialize-schema: always        # 生成 BATCH_* 元数据表（生产改 always → 手工建表）
    job:
      enabled: true                    # 启动时自动执行（一般设 false，手动触发）
  datasource: {url: ..., username: ..., password: ...}
```

```java
@SpringBootApplication
@EnableBatchProcessing
public class BatchApp {}
```

## 4. 一个完整 Job

```java
@Configuration
@RequiredArgsConstructor
public class OrderJobConfig {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager txManager;
    private final DataSource dataSource;

    @Bean
    public Job importOrderJob() {
        return new JobBuilder("importOrderJob", jobRepository)
                .start(importStep())
                .next(summaryStep())
                .listener(new JobExecutionListener() {
                    @Override public void afterJob(JobExecution e) {
                        log.info("job {} finished {}", e.getJobInstance().getJobName(), e.getStatus());
                    }
                })
                .build();
    }

    @Bean
    public Step importStep() {
        return new StepBuilder("importStep", jobRepository)
                .<OrderRow, Order>chunk(500, txManager)      // ★ 每 500 条一个事务
                .reader(reader())
                .processor(processor())
                .writer(writer())
                .faultTolerant()
                    .skip(ValidationException.class)
                    .skipLimit(100)
                    .retry(DeadlockLoserDataAccessException.class)
                    .retryLimit(3)
                .listener(new ItemReadListener<>() { /* ... */ })
                .build();
    }

    @Bean
    @StepScope                              // ★ 支持 jobParameters 延迟绑定
    public FlatFileItemReader<OrderRow> reader(@Value("#{jobParameters['file']}") String file) {
        return new FlatFileItemReaderBuilder<OrderRow>()
                .name("orderReader")
                .resource(new FileSystemResource(file))
                .delimited().delimiter(",")
                .names("id", "userId", "amount", "createTime")
                .targetType(OrderRow.class)
                .linesToSkip(1)             // 跳过表头
                .build();
    }

    @Bean
    public ItemProcessor<OrderRow, Order> processor() {
        return row -> new Order(row.getUserId(), new BigDecimal(row.getAmount()));
    }

    @Bean
    public JdbcBatchItemWriter<Order> writer() {
        return new JdbcBatchItemWriterBuilder<Order>()
                .dataSource(dataSource)
                .sql("insert into t_order(user_id, amount) values(:userId, :amount)")
                .beanMapped()
                .build();
    }
}
```

## 5. 常用 Reader / Writer

| Reader | 说明 |
|---|---|
| `FlatFileItemReader` | CSV/固定宽度文件 |
| `JdbcCursorItemReader` / `JdbcPagingItemReader` | 数据库游标/分页读 |
| `JpaPagingItemReader` | JPA 分页 |
| `StaxEventItemReader` | XML |
| `JsonItemReader` | JSON |
| `ListItemReader` | 内存列表（测试） |
| `KafkaItemReader` | Kafka |

| Writer | 说明 |
|---|---|
| `FlatFileItemWriter` | 文件 |
| `JdbcBatchItemWriter` | JDBC 批量 |
| `JpaItemWriter` | JPA |
| `CompositeItemWriter` | 组合 |
| `MultiResourceItemWriter` | 分片输出 |

## 6. 触发作业

```java
// 1 命令行
java -jar batch.jar --spring.batch.job.names=importOrderJob file=/data/orders.csv

// 2 REST 触发
@RestController
@RequiredArgsConstructor
public class JobController {
    private final JobLauncher jobLauncher;
    private final Job importOrderJob;

    @PostMapping("/jobs/import")
    public String run(@RequestParam String file) throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addString("file", file)
                .addLong("run.id", System.currentTimeMillis())   // ★ 让每次参数唯一，可重复执行
                .toJobParameters();
        return jobLauncher.run(importOrderJob, params).getStatus().name();
    }
}
```

> **JobParameters 相同则视为同一次执行**，成功后再次启动会报 `JobInstanceAlreadyCompleteException`。加时间戳可规避。

## 7. 故障处理

| 机制 | 配置 |
|---|---|
| 跳过 | `.faultTolerant().skip(XxxException.class).skipLimit(n)` |
| 重试 | `.retry(XxxException.class).retryLimit(n)` |
| 不回滚 | `.noRollback(XxxException.class)` |
| 重启 | 从上次失败的 chunk 继续（靠 `ExecutionContext` 保存读位置） |
| 停止 | `.listener(StepExecutionListener)` 自定义逻辑 |

```java
// 手动重启失败的 JobExecution
jobOperator.restart(failedExecutionId);
```

## 8. 并行与分片

| 方式 | 说明 |
|---|---|
| `Multi-threaded Step` | 单个 step 多线程（需 reader 线程安全） |
| `Parallel Steps` | 通过 `SplitBuilder` 并行多个无依赖 step |
| `Partitioning` | **主-从分片**（按数据范围分给多个 worker），最常用 |
| `Remote Chunking` | 通过 MQ 分发 chunk 到远程 worker |

```java
// 分片示例
@Bean
public Step masterStep() {
    return new StepBuilder("master", jobRepository)
            .partitioner("slaveStep", new SimplePartitioner())   // 按 ID 范围划分
            .step(slaveStep())
            .gridSize(10)
            .taskExecutor(new SimpleAsyncTaskExecutor("batch-"))
            .build();
}
```

## 9. 性能建议

- chunk 大小：500~1000（太小事务多，太大占用内存与锁）。
- 用 `JdbcPagingItemReader` 避免游标长时间占用连接。
- 批量插入开启 `rewriteBatchedStatements=true`。
- 关闭索引/约束（大批量导入时），或先导入再建索引。
- 只读事务 + 合理 fetchSize。

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 元数据表缺失 | 启动失败 | `initialize-schema: always` 或手工建表 |
| 相同参数无法重跑 | `JobInstanceAlreadyCompleteException` | 参数加 `run.id` |
| Reader 非线程安全 | 多线程数据错乱 | `synchronized` 或不用多线程 step |
| 大事务 | 锁超时 | 调小 chunk |
| 内存溢出 | 一次读全表 | 分页/游标读 |
| 重启从 0 开始 | 未保存状态 | 用支持 `ItemStream` 的 reader |
| 调度重复执行 | 多实例同时跑 | 分布式锁（ShedLock）或独立调度中心 |

## 11. 面试速答

- **Spring Batch 核心组件**：`Job` → `Step` → `ItemReader/Processor/Writer`，`JobRepository` 存元数据，`JobLauncher` 启动。
- **chunk 是什么**：读 N 条后一次性处理并写入，这 N 条作为一个事务提交；失败只回滚该 chunk。
- **失败如何续跑**：`ExecutionContext` 保存读取位置与状态，`JobOperator#restart` 从失败点继续。
- **并行方式**：多线程 step、并行 step、主-从分片（Partitioning）、远程分片。
- **与定时任务区别**：Batch 关注大数据量的可靠处理（事务、重启、跳过、监控）；`@Scheduled` 只是定时触发。
