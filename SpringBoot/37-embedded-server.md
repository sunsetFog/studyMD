# 37 · 内嵌容器：Tomcat 启动机制与调优

## 一、内嵌化的意义

```
传统：代码 → war → 部署到外置 Tomcat（共享容器，多应用混杂）
Boot：容器（Tomcat/Jetty/Undertow）是应用的一个 Bean，随应用启动/关闭
```

依赖切换：

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-web</artifactId>
  <exclusions><exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
  </exclusion></exclusions>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-undertow</artifactId>   <!-- 高并发可选 -->
</dependency>
```

| 容器 | 特点 |
|---|---|
| Tomcat（默认） | 生态稳，BIO→NIO |
| Jetty | 轻量，长连接场景好 |
| Undertow | 高并发吞吐好（XNIO），压测常胜出 |

## 二、启动机制（在 refresh 的 onRefresh 阶段）

```
ServletWebServerApplicationContext.onRefresh()
  → ServletWebServerFactoryAutoConfiguration
      → 注册 TomcatServletWebServerFactory Bean
  → onRefresh 创建 Tomcat 实例（嵌入式实例，非外部部署）
      → 配置 Connector（端口/协议）
      → 注册 DispatcherServlet（动态 Servlet）
  → finishRefresh 时 start()
      → Tomcat.start() 开始监听端口
      → WebServerStartStopLifecycle 打印 "Tomcat started on port 8080"
```

相关 Bean 链路：
- `ServerProperties` 绑定 `server.*` 配置
- `WebServerFactoryCustomizer` 集合：自定义容器参数的标准入口

```java
@Bean
public WebServerFactoryCustomizer<TomcatServletWebServerFactory> customizer() {
    return factory -> factory.addConnectorCustomizers(connector ->
            connector.setProperty("maxConnections", "10000"));
}
```

## 三、核心参数与调优

```yaml
server:
  port: 8080
  tomcat:
    threads:
      max: 200                # 最大工作线程（默认 200）
      min-spare: 10
    max-connections: 8192     # 最大连接数（默认 8192）
    accept-count: 100         # 连接队列长度（超过拒绝）
    connection-timeout: 20s
    max-http-form-post-size: 2MB
    uri-encoding: UTF-8
  compression:
    enabled: true             # 响应压缩
    min-response-size: 1KB
    mime-types: application/json,text/html
  shutdown: graceful
```

**调优公式**：
- `threads.max`：IO 密集 ≈ 200-500（线程不是越多越好，上下文切换有成本）
- 请求真正耗时在 DB → 瓶颈在连接池（09 篇 Hikari），线程数应与 DB 处理能力匹配
- QPS 估算：QPS ≈ threads.max × (1000/平均响应ms)

## 四、线程模型

```
Acceptor 线程      → 接受 TCP 连接
Poller 线程(NIO)   → 轮询就绪事件
工作线程池(200)    → 处理请求 → FilterChain → DispatcherServlet → Controller

同步阻塞链路：一个请求占一个工作线程直到响应完成
→ 慢 SQL/慢下游会耗尽线程池 → 排队 → accept-count 满 → 拒绝连接
```

**由此推出**：接口限流、超时控制（连接层 + HTTP 客户端超时）、线程池隔离（19 篇）都是防止雪崩的关键。

## 五、虚拟线程（Java 21 + Boot 3.2+）

```yaml
spring:
  threads:
    virtual:
      enabled: true    # Tomcat 工作线程改为虚拟线程
```

阻塞 IO 场景吞吐大幅提升，代码无感改造。注意：ThreadLocal 兼容性、synchronized 阻塞 pinning 问题、连接池本身仍是瓶颈。

## 六、与外部负载均衡的协作

```
客户端 → Nginx/K8s Ingress → app(8080) × N 实例
```

- `server.forward-headers-strategy: framework` 信任 X-Forwarded-*（获取真实 IP/协议）
- 优雅停机（30 篇）+ 摘流量窗口（31 篇 preStop）
- WebSocket 需要 Nginx Upgrade 头（28 篇）

## 七、问题排查命令

```bash
# 线程池饱和：大量 http-nio-8080-exec-* WAITING
jstack <pid> | grep 'http-nio' | awk '{print $2}' | sort | uniq -c

# 连接数
ss -tan | grep 8080 | awk '{print $1}' | sort | uniq -c

# 火焰图定位热点
async-profiler / Arthas
```

## 常见坑

- **坑 1**：threads.max 调到 2000 以为抗并发 → 每线程 1MB 栈内存 + 上下文切换，反而更慢；先优化慢调用
- **坑 2**：max-connections 与 accept-count 配错 → 高峰全量拒绝，错误日志却看不到（连接层拒绝）
- **坑 3**：上传大小只改 multipart 没改 tomcat max-http-form-post-size → 表单仍报错
- **坑 4**：内网多实例没配 forward-headers → 日志/限流记录的 IP 全是 LB 的
- **坑 5**：虚拟线程下使用了重 synchronized 的老代码 → pinning 吞吐不升反降

## Checklist

- [ ] 容器线程参数按压测结果而非拍脑袋设置
- [ ] 超时/限流/线程池隔离防雪崩
- [ ] 压缩开启，JSON 响应收益明显
- [ ] Java 21 项目评估虚拟线程
