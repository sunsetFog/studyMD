# 25 · 虚拟线程（Java 21）与结构化并发

## 一、为什么需要虚拟线程

**平台线程（传统 Thread）**：1:1 映射 OS 线程，每个占 ~1MB 栈，创建/调度昂贵。IO 密集场景线程大部分时间阻塞等待 → 线程池开多大都浪费。

**虚拟线程**：JVM 管理的轻量线程（Java 21 正式版）， Millions 级可创建。阻塞 IO 时**自动让出载体线程**（unmount），IO 完成再挂回——用同步阻塞的代码风格获得异步的吞吐。

| 维度 | 平台线程 | 虚拟线程 |
|---|---|---|
| 创建成本 | ~1MB 栈 + 内核对象 | 几 KB，堆上对象 |
| 数量级 | 数千 | 百万 |
| 阻塞 IO | 占住线程（昂贵） | 让出（廉价） |
| CPU 密集 | 无差别 | 无优势 |
| 需要 | 线程池 | **不需要池** |

## 二、基本用法

```java
// 1. 直接创建
Thread.startVirtualThread(() -> handle(request));

// 2. Builder（更常用）
Thread v = Thread.ofVirtual().name("vt-", 0).start(() -> handle());
v.join();

// 3. Executor（API 兼容：接入现有池化代码）
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> callApi());       // 每任务一个虚拟线程，无池化复用
}

// 4. SpringBoot 3.2+ 一行开启（衔接 SpringBoot 系列 37 篇）
// spring.threads.virtual.enabled=true → Tomcat 工作线程变虚拟线程
```

## 三、适用与不适用

```
✓ 适合：IO 密集高并发（HTTP 调用、DB 查询、文件、RPC 中转）
✗ 不适合：
  - CPU 密集计算（核数就那么多，虚拟线程无增益）
  - 依赖 ThreadLocal 大对象的场景（百万线程 × 大对象 = OOM）
  - 需要 Pinning 的 synchronized 大临界区（见下）
```

## 四、Pinning（钉住）陷阱

虚拟线程阻塞时挂到「载体线程」（平台线程）上。以下情况会 **pin 住载体线程无法让出**：

```java
// 1. synchronized 块内阻塞（老 JDK 实现）
synchronized (lock) {
    socket.read();        // 阻塞期间载体线程被占住 → 吞吐退化回平台线程模式
}
// Java 24+ 已修复大部分 pinning；21 上规避：改用 ReentrantLock

// 2. native 方法/JNI 内阻塞
```

**迁移建议**：IO 密集服务里的长临界区 synchronized 改 ReentrantLock。

## 五、结构化并发（Preview，理念重要）

```java
// 问题：普通并发任务生命周期失控（孤儿线程、泄漏）
// 结构化并发：子任务生命周期绑定父作用域，取消传播、错误聚合
try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
    var user = scope.fork(() -> fetchUser(id));
    var orders = scope.fork(() -> fetchOrders(id));
    scope.join().throwIfFailed();             // 一个失败全部取消
    return new Data(user.get(), orders.get());
}
// 语义：任务树像代码块一样结构化——进入/退出即启动/回收
```

Java 21 为 preview API，生产等正式化；但「作用域管理任务生命周期」的设计思想是未来方向。

## 六、迁移实战 checklist（IO 密集服务）

```java
// 1. 线程池配置的重新思考
// 旧：executor 线程数 = 核数×2（因为阻塞贵）
// 新：虚拟线程无池化，并发上限改由 Semaphore/连接池控制
Semaphore dbGate = new Semaphore(50);          // 真正的资源上限：DB 连接
try (var ex = Executors.newVirtualThreadPerTaskExecutor()) {
    for (var req : requests) {
        ex.submit(() -> {
            dbGate.acquire();
            try { db.query(req); } finally { dbGate.release(); }
        });
    }
}

// 2. 瓶颈转移：线程不再是瓶颈，DB 连接池/下游限流才是 → 信号量/资源池限流

// 3. 观测：线程名改为任务语义；jstack 平台线程少了，虚拟线程用新工具看
```

## 七、与 CompletableFuture 的关系

| | CompletableFuture | 虚拟线程 |
|---|---|---|
| 风格 | 回调链式 | 同步直写 |
| 可读性 | 链长难读 | 顺序代码 |
| 组合能力 | allOf/anyOf 强 | 结构化并发（preview） |
| 适用 | 已有异步生态/纯回调场景 | 新 IO 密集代码首选 |

**趋势**：同步风格 + 虚拟线程正在取代回调式异步编排的多数场景。

## 常见坑

- **坑 1**：给虚拟线程建池 → 违背设计初衷；每任务一线程
- **坑 2**：synchronized 长阻塞 pinning（21 版本）→ ReentrantLock 替代
- **坑 3**：ThreadLocal 大对象 × 百万线程 → OOM；作用域值（ScopedValue）是未来替代
- **坑 4**：CPU 密集任务迁虚拟线程 → 无收益
- **坑 5**：以为不用管限流 → 并发上限转移到下游资源（连接池/信号量）

## Checklist

- [ ] 明确场景是 IO 密集才启用
- [ ] 每任务一线程，不池化
- [ ] 资源上限用 Semaphore/连接池表达
- [ ] synchronized 长临界区改为 ReentrantLock
- [ ] SpringBoot 3.2+ 评估 virtual.enabled
