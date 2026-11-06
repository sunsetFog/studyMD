# 34 · GraalVM 原生镜像与 AOT

## 一、动机

JVM 模式痛点：启动 10-30s、内存 500MB+、需要预热。**Native Image** 把应用 AOT 编译为独立可执行文件：

| 维度 | JVM 模式 | Native Image |
|---|---|---|
| 启动 | 10-30s | **< 0.5s** |
| 内存 | 500MB-1G | **100-200MB** |
| 吞吐峰值 | 高（JIT） | 略低（无 JIT，约低 10-30%） |
| 构建时间 | 快 | 慢（分钟级） |
| 反射/动态代理 | 自由 | 需显式配置 |
| 调试/监控 | 生态全 | 受限 |

适用：Serverless/FaaS、容器弹性伸缩频繁、CLI 工具。**长驻大流量后端服务仍推荐 JVM 模式**。

## 二、构建原生镜像

### 方式 1：本地构建（需 GraalVM + C 工具链）

```bash
# 前置：GraalVM JDK 17+、xcode build tools（mac）/ gcc（linux）
mvn -Pnative native:compile
./target/demo-app     # 直接运行的可执行文件
```

```xml
<profiles>
  <profile>
    <id>native</id>
    <build>
      <plugins>
        <plugin>
          <groupId>org.graalvm.buildtools</groupId>
          <artifactId>native-maven-plugin</artifactId>
        </plugin>
      </plugins>
    </build>
  </profile>
</profiles>
```

### 方式 2：Docker 构建（推荐，免本地工具链）

```bash
docker run --rm -v "$PWD":/work -w /work \
  ghcr.io/graalvm/native-image-community:17 \
  mvn -Pnative native:compile
```

或 SpringBoot 官方 Buildpacks 一条命令：

```bash
mvn spring-boot:build-image -Pnative
```

## 三、AOT 限制与配置

Native Image 默认**封闭世界假设**：编译期不可见的反射/代理/资源都会失败。

SpringBoot 3 的解决路径：
1. **自动 AOT 处理**：Spring AOT 引擎在构建期分析 Bean 定义，生成可达性配置（大部分常规应用零配置）
2. **手动 hints**（自定义反射/序列化场景）：

```java
@Component
public class MyRuntimeHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader loader) {
        hints.reflection().registerType(UserDTO.class,
                MemberCategory.DECLARED_FIELDS,
                MemberCategory.INVOKE_DECLARED_METHODS);
        hints.resources().registerPattern("mapper/*.xml");
        hints.serialization().registerType(LocalDateTime.class);
    }
}
```

3. 运行时探测：`-agentlib:native-image-agent=config-output-dir=META-INF/native-image` 在 JVM 模式运行并记录所需配置。

**常见踩雷点**：
- MyBatis XML mapper + 动态代理 → 需 hints 注册；MyBatis 官方有 native 支持分支，复杂动态 SQL 谨慎
- Jackson 对未注册 DTO → 自动处理了常见场景，自定义 Serializer 要 hints
- Lombok 无影响（编译期）
- CGLIB 代理在 native 不可用 → 用 JDK 接口代理

## 四、CRaC 另一选项（简化）

**CRaC（Coordinated Restore at Checkpoint）**：JVM 模式下对预热后的进程做快照，恢复时毫秒级启动——兼顾 JIT 性能与快速启动：

```bash
# checkpoint 后 crac-engine 恢复
java -XX:CRaCCheckpointTo=/checkpoint -jar app.jar
jcmd app.jar JDK.checkpoint
```

适合已深度依赖 JVM 生态（如 MyBatis 复杂栈）又需要快速扩容的场景。

## 五、选型决策

```
Serverless / CLI / 边缘部署          → Native Image
需要毫秒级扩容 + JIT 性能            → CRaC
长驻常规后端服务（99% 场景）         → JVM 模式 + 容器 + 预热
```

## 常见坑

- **坑 1**：native 构建本地缺工具链失败 → 用 Docker 方式构建
- **坑 2**：运行时 NoSuchMethodError/反射失败 → 补 hints 或用 agent 生成配置
- **坑 3**：native 下内存占用虽低但 GC 停顿模式不同 → 峰值场景压测后再上
- **坑 4**：静态初始化在构建期执行产生意外结果 → @AutomaticTargetBuildTime 初始化控制

## Checklist

- [ ] 明确是否真的需要 native（多数不需要）
- [ ] 常规应用先验证 Spring AOT 自动处理覆盖率
- [ ] 自定义反射/资源补 RuntimeHints
- [ ] native 构建接入 CI（Docker 构建）
