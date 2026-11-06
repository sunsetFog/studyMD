# 57 · gRPC 与 Protobuf

## 1. 为什么用 gRPC

| 对比 | HTTP/JSON | gRPC |
|---|---|---|
| 协议 | HTTP/1.1（文本） | **HTTP/2**（二进制） |
| 序列化 | JSON | **Protobuf**（紧凑、快） |
| 契约 | OpenAPI（可选） | `.proto` **强契约**，生成代码 |
| 性能 | 一般 | 高（3~10 倍） |
| 流式 | 难 | 支持四种流 |
| 浏览器 | 原生支持 | 需 grpc-web |
| 可读性 | 高 | 低（二进制，调试不便） |

适用：**内部服务间高频调用**、多语言互通、需要流式（如实时推送）。

## 2. Protobuf 基础

```protobuf
syntax = "proto3";

package com.example.order;

option java_multiple_files = true;
option java_package = "com.example.grpc.order";
option java_outer_classname = "OrderProto";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (OrderReply);
  rpc CreateOrder (CreateOrderRequest) returns (OrderReply);
  rpc ListOrders (ListRequest) returns (stream OrderReply);      // 服务端流
  rpc BatchCreate (stream CreateOrderRequest) returns (BatchReply); // 客户端流
  rpc SyncOrders (stream SyncRequest) returns (stream SyncReply);   // 双向流
}

message GetOrderRequest {
  int64 id = 1;                    // ★ 数字是字段编号，不是值
}

message OrderReply {
  int64 id = 1;
  int64 user_id = 2;
  string order_no = 3;
  double amount = 4;               // 金额建议用 string 或 int64(分)
  OrderStatus status = 5;
  repeated OrderItem items = 6;    // 列表
  map<string, string> ext = 7;
  google.protobuf.Timestamp create_time = 8;
}

enum OrderStatus {
  UNKNOWN = 0;                     // ★ proto3 枚举第一个必须为 0
  CREATED = 1;
  PAID = 2;
}
```

要点：
- 字段编号 **1~15 只占 1 字节**，留给高频字段。
- 编号**不能改**（改了不兼容）；删除字段用 `reserved 5;`。
- 默认值不传输（无法区分「未设置」与「默认值」，proto3 可用 `optional` 或 wrapper 类型）。

## 3. 代码生成（Maven 插件）

```xml
<build>
  <extensions>
    <extension><groupId>kr.motd.maven</groupId><artifactId>os-maven-plugin</artifactId>
               <version>1.7.1</version></extension>
  </extensions>
  <plugins>
    <plugin>
      <groupId>org.xolstice.maven.plugins</groupId>
      <artifactId>protobuf-maven-plugin</artifactId>
      <version>0.6.1</version>
      <configuration>
        <protocArtifact>com.google.protobuf:protoc:3.25.1:exe:${os.detected.classifier}</protocArtifact>
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.60.0:exe:${os.detected.classifier}</pluginArtifact>
      </configuration>
      <executions><execution><goals><goal>compile</goal><goal>compile-custom</goal></goals></execution></executions>
    </plugin>
  </plugins>
</build>
```

生成到 `target/generated-sources/protobuf/`，Maven 编译时自动执行。

## 4. Spring Boot 集成（grpc-java + 自定义 starter 或 net.devh）

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>2.15.0.RELEASE</version>
</dependency>
<dependency><groupId>net.devh</groupId>
           <artifactId>grpc-client-spring-boot-starter</artifactId>
           <version>2.15.0.RELEASE</version></dependency>
```

### 服务端

```yaml
grpc:
  server:
    port: 9090          # HTTP/2 端口（与 HTTP 端口不同）
```
```java
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    @Override
    public void getOrder(GetOrderRequest req, StreamObserver<OrderReply> observer) {
        try {
            Order o = service.findById(req.getId());
            observer.onNext(toReply(o));
            observer.onCompleted();          // ★ 必须调用
        } catch (Exception e) {
            observer.onError(Status.NOT_FOUND.withDescription("订单不存在").asRuntimeException());
        }
    }

    @Override
    public void listOrders(ListRequest req, StreamObserver<OrderReply> observer) {
        service.stream(req).forEach(o -> observer.onNext(toReply(o)));
        observer.onCompleted();              // 服务端流：多次 onNext + 一次 onCompleted
    }
}
```

### 客户端

```yaml
grpc:
  client:
    order-service:
      address: static://localhost:9090
      # 或用注册中心：discovery://order-service（配合 Nacos/Eureka）
      enable-keep-alive: true
      negotiation-type: plaintext           # 生产用 TLS
```
```java
@Service
public class OrderClient {
    @GrpcClient("order-service")
    private OrderServiceGrpc.OrderServiceBlockingStub stub;    // 同步
    // 或 OrderServiceStub（异步）、OrderServiceFutureStub（Future）

    public OrderReply get(Long id) {
        return stub.getOrder(GetOrderRequest.newBuilder().setId(id).build());
    }
}
```

## 5. 四种通信模式

| 模式 | 定义 | 场景 |
|---|---|---|
| Unary | 一请求一响应 | 普通调用 |
| Server streaming | 一请求多响应 | 拉取大量数据、订阅 |
| Client streaming | 多请求一响应 | 批量上传 |
| Bidirectional | 双向流 | 实时通信、消息推送 |

## 6. 拦截器（认证、日志、链路）

```java
@GrpcGlobalServerInterceptor
public class LogInterceptor implements ServerInterceptor {
    @Override public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call, Metadata headers, ServerCallHandler<ReqT, RespT> next) {
        log.info("method={}", call.getMethodDescriptor().getFullMethodName());
        return next.startCall(call, headers);
    }
}

// 客户端传递 token
@GrpcGlobalClientInterceptor
public class AuthInterceptor implements ClientInterceptor {
    @Override public <Q,S> ClientCall<Q,S> interceptCall(MethodDescriptor<Q,S> m, CallOptions o, Channel c) {
        return new ForwardingClientCall.SimpleForwardingClientCall<>(c.newCall(m, o)) {
            @Override public void start(Listener<S> l, Metadata h) {
                h.put(Metadata.Key.of("Authorization", Metadata.ASCII_STRING_MARSHALLER), "Bearer " + token());
                super.start(l, h);
            }
        };
    }
}
```

## 7. 错误处理

状态码：`OK`、`CANCELLED`、`NOT_FOUND`、`ALREADY_EXISTS`、`PERMISSION_DENIED`、`RESOURCE_EXHAUSTED`（限流）、`UNAUTHENTICATED`、`UNAVAILABLE`（重试）、`DEADLINE_EXCEEDED`、`INTERNAL`。

```java
// 客户端重试（可重试状态码）
ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
        .enableRetry()
        .maxRetryAttempts(3)
        .defaultServiceConfig(config)
        .build();
```

## 8. 与 Spring Cloud 集成

- **服务发现**：`net.devh` starter 支持 `discovery://order-service`（Nacos/Eureka/Consul）。
- **负载均衡**：gRPC 客户端自带（RoundRobin/PickFirst）。
- **链路追踪**：需拦截器传递 `traceparent` 头。
- **网关**：Gateway 不支持 gRPC 转发（可用 Envoy / grpc-gateway / APISIX）。

## 9. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 与 HTTP 端口冲突 | 启动失败 | gRPC 用独立端口 |
| 浏览器直连失败 | 不支持 HTTP/2 h2c | 用 grpc-web + Envoy 代理 |
| 金额精度 | double 失真 | 用 `string` 或 `int64`（分） |
| proto 变更不兼容 | 解析失败 | 只加字段不改编号，删字段用 reserved |
| 超时未设 | 一直等 | `stub.withDeadlineAfter(3, SECONDS)` |
| `onCompleted` 未调 | 客户端一直阻塞 | 必须调用 |
| 阻塞 stub 在响应式里用 | 阻塞 EventLoop | 用 `FutureStub` 或线程池 |
| 服务发现地址格式 | 连不上 | `discovery://` 需要注册中心地址 |

## 10. 面试速答

- **gRPC 为什么快**：HTTP/2（多路复用、头部压缩、二进制帧）+ Protobuf 紧凑二进制序列化 + 代码生成避免反射。
- **Protobuf 与 JSON**：Protobuf 体积小 3~10 倍、解析快，但需要 `.proto` 契约与生成代码，可读性差。
- **四种流**：Unary、服务端流、客户端流、双向流。
- **HTTP/2 多路复用**：同一 TCP 连接上并行多个 Stream，解决队头阻塞。
- **什么时候不用 gRPC**：对外 API（浏览器/第三方）用 HTTP/JSON；需要人类可读调试的场景。
