# 16 · Bean 生命周期与作用域

## 一、生命周期全景

```
1. 实例化 Instantiation        （反射调构造器，得到原始对象）
2. 属性填充 Populate           （依赖注入 @Autowired 生效）
3. Aware 回调                  （BeanNameAware → BeanFactoryAware → ApplicationContextAware）
4. BeanPostProcessor#postProcessBeforeInitialization
5. 初始化 Initialization       （@PostConstruct → InitializingBean#afterPropertiesSet → initMethod）
6. BeanPostProcessor#postProcessAfterInitialization   （AOP 代理在这里生成！）
7. 就绪使用
8. 容器关闭                    （@PreDestroy → DisposableBean#destroy → destroyMethod）
```

**关键结论**：`@Autowired` 在 `@PostConstruct` 之前完成 → 初始化逻辑放心用注入的依赖。

## 二、常用钩子代码

```java
@Component
public class CacheWarmer implements InitializingBean, DisposableBean {

    @PostConstruct                       // 推荐：标准 JSR-250
    public void postConstruct() {
        System.out.println("依赖注入完成，可做预热");
    }

    @Override
    public void afterPropertiesSet() { }  // Spring 接口方式（耦合框架，少用）

    @PreDestroy
    public void preDestroy() {            // 优雅关闭：释放连接/停线程池
        System.out.println("容器关闭前清理");
    }

    @Override
    public void destroy() { }
}
```

XML/@Bean 方式指定方法：

```java
@Bean(initMethod = "start", destroyMethod = "stop")
public Server server() { return new Server(); }
```

## 三、BeanPostProcessor（扩展的钥匙）

所有 Bean 都会经过，**全局增强用**：

```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String name) {
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        if (bean instanceof CacheableService) {
            // 包装为代理……
        }
        return bean;
    }
}
```

Spring 自身的诸多能力都基于它：@Autowired 处理（AutowiredAnnotationBeanPostProcessor）、AOP 代理（AbstractAutoProxyCreator）、@Configuration 增强、@Async 支持。

## 四、BeanFactoryPostProcessor

在 Bean **实例化之前**修改 BeanDefinition——比 BeanPostProcessor 更早：

```java
@Component
public class MyBFPP implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory factory) {
        BeanDefinition bd = factory.getBeanDefinition("orderService");
        bd.setInitMethodName("customInit");
    }
}
```

`@ConfigurationProperties` 绑定、占位符解析都由 BFPP 完成。

## 五、初始化顺序控制

```java
@DependsOn("dataSourceInit")     // 显式依赖
@Order(1)                        // 集合注入/监听器顺序，不控制 Bean 创建顺序
```

## 六、@PostConstruct 实战场景

| 场景 | 示例 |
|---|---|
| 缓存预热 | 启动时加载字典表到本地缓存 |
| 连接建立 | 启动时校验第三方服务可达 |
| 配置校验 | 启动时断言关键配置合法，快速失败 |
| 状态机初始化 | 构建 Map 常量 |

> 注意：@PostConstruct 里只做轻量初始化；耗时操作放 `ApplicationRunner`/`CommandLineRunner`（36 篇），避免拖慢启动且此时服务未就绪，不影响流量。

```java
@Component
public class StartupRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) { /* 容器就绪后执行 */ }
}
```

## 七、单例 vs 原型生命周期差异

- singleton：容器启动创建、容器关闭销毁、容器负责全部生命周期
- prototype：注入/获取时创建，**销毁由使用者负责**，@PreDestroy 不被调用

## 常见坑

- **坑 1**：@PostConstruct 里抛异常 → 应用启动失败（合理利用做快速失败）
- **坑 2**：在 @PostConstruct 调用其他 Bean 的异步方法/发布事件，部分 Bean 尚未就绪 → 用 ApplicationRunner
- **坑 3**：构造器里用 @Autowired 字段 → 还是 null（注入发生在构造之后）
- **坑 4**：BeanPostProcessor 处理太早的 Bean 时依赖注入自身异常 → BPP 支持静态注册/early 感知

## Checklist

- [ ] 初始化逻辑统一 @PostConstruct（轻量）或 ApplicationRunner（耗时）
- [ ] @PreDestroy 释放资源，支持优雅停机
- [ ] 理解 AOP 代理生成时机（postProcessAfterInitialization）
- [ ] 全局扩展需求用 BeanPostProcessor 而非散落的工具方法
