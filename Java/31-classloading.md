# 31 · 类加载机制

## 一、类的生命周期

```
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
```

| 阶段 | 做什么 |
|---|---|
| 加载 | 读 .class 字节流 → 方法区元数据 → 生成 Class 对象 |
| 验证 | 格式/语义/字节码安全校验 |
| 准备 | **静态变量分配内存并设零值**（final 常量除外，此时直接赋值） |
| 解析 | 符号引用 → 直接引用 |
| 初始化 | **执行静态变量显式赋值 + static 代码块（`<clinit>`）** |

```java
// 准备 vs 初始化的经典题
static int a = 1;
// 准备阶段：a = 0
// 初始化阶段：a = 1（<clinit> 里赋值）

static final int CONST = 10;
// 编译期常量：准备阶段即 10（ConstantValue 属性）
```

## 二、初始化触发时机（主动引用）

```
✓ new / getstatic / putstatic / invokestatic（读静态变量除外——只读不触发）
✓ 反射 Class.forName
✓ 初始化子类 → 父类先初始化
✓ main 所在主类
✓ Lambda/MethodHandle 相关

✗ 不触发（被动引用）：
- 通过子类引用父类静态字段 → 只初始化父类
- 定义类数组（ClassName[] arr）
- 引用编译期常量
```

## 三、类加载器与双亲委派

```
Bootstrap ClassLoader      ← 核心类库（rt.jar/java.base），C++ 实现
    ↑ 委派
Platform ClassLoader       ← 平台类（旧 ExtClassLoader）
    ↑ 委派
Application ClassLoader    ← classpath 应用类
    ↑ 委派
自定义 ClassLoader         ← 热部署/隔离/加密
```

### 双亲委派模型

```
收到加载请求 → 先委派给父加载器 → 父无法加载才自己加载
```

**意义**：
1. 防止核心类被篡改（自定义 java.lang.String 永远加载不到）
2. 同一类全局唯一（安全 + 一致性）

**打破双亲委派的场景**：SPI 机制（JDBC 驱动由线程上下文类加载器反向加载）、OSGi/Tomcat 应用隔离（每个 webapp 独立加载器）、热部署（丢弃旧加载器换新）。

## 四、自定义类加载器

```java
public class MyLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = loadBytes(name);        // 从自定义源（文件/网络/解密）读字节
        return defineClass(name, bytes, 0, bytes.length);
    }
}

Class<?> c = new MyLoader(parent).loadClass("com.example.Plugin");
```

## 五、类卸载条件（三个同时满足）

```
1. 该类所有实例已被回收
2. Class 对象无引用
3. 类加载器已被回收
```

→ **类加载器泄漏是 Metaspace OOM 的主因**：静态集合持有对象 → Class 无法卸载。

## 六、静态初始化的线程安全

```java
// <clinit> 由 JVM 保证线程安全且仅执行一次 → 静态内部类单例的基础
class Singleton {
    private Singleton() { }
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();   // 首次引用 Holder 才初始化
    }
    public static Singleton getInstance() { return Holder.INSTANCE; }
}
// 惰性 + 线程安全 + 无锁 —— 类加载机制白送
```

## 七、模块系统（JPMS，Java 9+）速览

```java
module com.example.app {
    requires com.example.lib;      // 显式依赖
    exports com.example.app.api;   // 只导出 API 包
    opens com.example.app.model to com.fasterxml.jackson.databind;  // 反射开放给指定模块
}
// 未 exports/opens 的包 → 反射 InaccessibleObjectException
```

## 常见坑

- **坑 1**：static 字段顺序依赖（static int a = b; static int b = 1） → 前向引用错乱
- **坑 2**：静态集合缓存导致类加载器泄漏 → Metaspace OOM
- **坑 3**：NoClassDefFoundError（编译有运行无/初始化失败） vs ClassNotFoundException（找不到类）区别
- **坑 4**：类初始化死锁（两个类互相在 <clinit> 中触发对方）
- **坑 5**：依赖冲突导致 NoSuchMethodError → mvn dependency:tree 排除（38 篇）

## Checklist

- [ ] 说清五个阶段与准备/初始化的赋值差异
- [ ] 双亲委派模型及打破场景
- [ ] 静态内部类单例的原理
- [ ] 类卸载三条件与泄漏路径
