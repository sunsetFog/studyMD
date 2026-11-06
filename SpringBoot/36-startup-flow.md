# 36 · 启动流程：从 main 到容器就绪

## 一、总流程图

```
main()
 └─ SpringApplication.run()
     ├─ 1. 创建 SpringApplication 实例（推断应用类型/加载 Initializer & Listener）
     ├─ 2. 准备环境 Environment（加载 application.yml、Profile、命令行）
     ├─ 3. 创建 ApplicationContext（按类型选 AnnotationConfigServletWebServerApplicationContext）
     ├─ 4. prepareContext：注册主类为 BeanDefinition
     ├─ 5. refreshContext：容器刷新（Spring 核心流程，见下）
     ├─ 6. 自动配置在 refresh 中触发（ConfigurationClassPostProcessor）
     ├─ 7. 内嵌 Web 服务器启动（37 篇）
     └─ 8. 回调 Runners（ApplicationRunner / CommandLineRunner）
```

## 二、各阶段详解

### 阶段 1：SpringApplication 构造

```java
new SpringApplication(Main.class) 时：
- 推断应用类型：NONE / SERVLET / REACTIVE（classpath 有没有 spring-webmvc/webflux）
- 从 spring.factories 加载 ApplicationContextInitializer 与 ApplicationListener
- 推断 main class
```

### 阶段 2：Environment 准备

```
收集 property sources（按 03 篇优先级）：
命令行 > 环境变量 > 外置 yml > jar 内 yml > 默认值
→ 激活 Profile
→ 触发 ApplicationEnvironmentPreparedEvent
```

### 阶段 3-4：容器创建与主类注册

- REACTIVE → AnnotationConfigReactiveWeb...；SERVLET → AnnotationConfigServletWeb...；普通 → AnnotationConfig...
- 主启动类被注册为 BeanDefinition（配置类身份）

### 阶段 5：refresh() —— Spring 的心脏

```
invokeBeanFactoryPostProcessors
  ├─ ConfigurationClassPostProcessor 解析配置类
  │    → 处理 @ComponentScan 扫描业务 Bean
  │    → 处理 @Import → AutoConfigurationImportSelector（35 篇自动配置）
  │    → @Bean 方法注册
  └─ 其他 BFPP（属性占位符解析、@ConfigurationProperties 绑定）

registerBeanPostProcessors       // 注册 BPP（@Autowired、AOP 代理靠它们）

onRefresh                        // 子类扩展点：Servlet 容器在此创建（37 篇）

finishBeanFactoryInitialization  // 实例化全部非懒加载单例 Bean
                                 // （16 篇 Bean 生命周期在此发生）
```

**信息闭环**：为什么启动类必须在根包？——ComponentScan 从主类包开始；为什么改配置能生效？——Environment 在 Bean 实例化前就绪。

## 三、生命周期事件时序

```
ApplicationStartingEvent          → 最早，Environment 未就绪
ApplicationEnvironmentPreparedEvent
ApplicationPreparedEvent          → 容器创建未刷新
ApplicationStartedEvent           → refresh 完成，Runners 之前
ApplicationReadyEvent             → 全部就绪，可接收流量（监控埋点最佳时机）
ApplicationFailedEvent            → 启动失败（告警）
```

```java
@EventListener(ApplicationReadyEvent.class)
public void onReady() { log.info("服务就绪，开始预热缓存"); }
```

## 四、启动加速手段

| 手段 | 效果 |
|---|---|
| 懒加载 `spring.main.lazy-initialization=true` | 启动快，首请求慢（谨慎：错误延迟暴露） |
| 减少 @ComponentScan 范围 | 扫描是启动大头 |
| CDS / AppCDS | 类加载缓存，JVM 模式提速 30-40% |
| Spring AOT + Native | 极致（34 篇） |
| 排除不需要的自动配置 `spring.autoconfigure.exclude` | 少走条件判断 |

## 五、启动失败的排查路径

```
1. 读启动日志第一个 ERROR（通常是根因）
2. BeanCurrentlyInCreationException   → 循环依赖（15 篇）
3. NoSuchBeanDefinitionException      → 缺依赖/没扫描到/条件没满足（35 篇 debug:true）
4. BeanDefinitionOverrideException    → Bean 重名（allow-bean-definition-overriding 不要全局开）
5. UnsatisfiedDependencyException     → 循追溯因链，最底是真实缺失的依赖
6. Port already in use                → 37 篇
7. Flyway 校验失败                    → V 版本号冲突
```

## 六、优雅的启动钩子选择

| 钩子 | 时机 | 适用 |
|---|---|---|
| @PostConstruct | Bean 初始化 | 轻量单 Bean 初始化 |
| @EventListener(ApplicationReadyEvent) | 全部就绪 | 跨 Bean 预热/上报 |
| ApplicationRunner | Runners 阶段 | 启动任务（如拉取配置校验） |
| SmartLifecycle.start | 容器生命周期 | 有启停语义的组件 |

## 常见坑

- **坑 1**：启动钩子里做网络请求阻塞启动 → 超时兜底或异步化
- **坑 2**：lazy-initialization 全局开启 → 配置错误运行时才炸；只对特定 Bean 用 @Lazy
- **坑 3**：ApplicationRunner 抛异常 → 整个应用启动失败（合理：快速失败；注意别把非关键任务放这里）
- **坑 4**：监听 StartingEvent 想拿配置 → Environment 未就绪，用 EnvironmentPreparedEvent

## Checklist

- [ ] 能讲清 run() 六步与 refresh 内的自动配置触发点
- [ ] 启动钩子按轻重选择正确时机
- [ ] 启动失败排查有固定路径（根因定位）
- [ ] 启动耗时达标（监控 ReadyEvent 时间戳）
