# 49 · 类加载机制

## 1. 类的生命周期

```
加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载
 └────────── 连接（Linking）──────────┘
```

### 加载（Loading）

1. 通过全限定名获取二进制字节流（磁盘、网络、动态生成）。
2. 转为方法区的运行时数据结构。
3. 在堆中生成 `Class` 对象作为访问入口。

### 验证（Verification）

- 文件格式验证（魔数 `CAFEBABE`、版本号）。
- 元数据验证（父类、接口、字段）。
- 字节码验证（控制流、类型转换）。
- 符号引用验证（引用的类/方法是否存在）。

### 准备（Preparation）

为静态变量分配内存并设**零值**（不是代码中的初始值）。

```java
static int value = 100;        // 准备阶段 value = 0，初始化阶段才 = 100
static final int CONST = 100;  // 编译期常量，准备阶段直接 = 100
```

### 解析（Resolution）

符号引用 → 直接引用（内存地址）。可延迟到首次使用时。

### 初始化（Initialization）

执行 `<clinit>()` 方法（静态变量赋值 + 静态块），由 JVM 保证线程安全。

**主动使用触发初始化**（6 种）：

1. `new`、读写静态字段（非 final 常量）、调用静态方法。
2. 反射调用（`Class.forName`）。
3. 初始化子类时先初始化父类。
4. 虚拟机启动时的主类（含 `main`）。
5. `MethodHandle`/`VarHandle` 解析结果。
6. 接口的默认方法被实现类调用（接口需初始化）。

**被动引用不触发**：

```java
// 1. 通过子类访问父类静态字段 → 只初始化父类
System.out.println(Sub.parentValue);

// 2. 数组定义 → 不初始化元素类
Sub[] arr = new Sub[10];

// 3. 访问编译期常量 → 不初始化（常量已内联到调用方）
System.out.println(Sub.CONST);
```

## 2. 类加载器

| 加载器 | 实现 | 加载路径 |
|---|---|---|
| Bootstrap | C++（HotSpot） | `$JAVA_HOME/lib`（rt.jar/modules） |
| Extension/Platform | Java | `$JAVA_HOME/lib/ext`（JDK 9+ 为 platform 模块） |
| Application/System | Java | `-classpath` |
| 自定义 | Java | 自定义路径 |

```java
ClassLoader cl = MyClass.class.getClassLoader();
System.out.println(cl);                              // AppClassLoader
System.out.println(cl.getParent());                  // PlatformClassLoader
System.out.println(cl.getParent().getParent());      // null（Bootstrap）

String.class.getClassLoader();                       // null（Bootstrap 加载）
```

## 3. 双亲委派模型

```
AppClassLoader.loadClass(name)
  ├── 已加载 → 返回
  └── 委派 parent.loadClass(name)
        └── Platform → Bootstrap
              └── 找不到 → 回退到子加载器尝试
```

```java
protected Class<?> loadClass(String name, boolean resolve) {
    synchronized (getClassLoadingLock(name)) {
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            try {
                if (parent != null) c = parent.loadClass(name, false);
                else c = findBootstrapClassOrNull(name);
            } catch (ClassNotFoundException ignored) { }
            if (c == null) c = findClass(name);      // 自己加载
        }
        if (resolve) resolveClass(c);
        return c;
    }
}
```

### 为什么需要双亲委派

1. **安全**：防止自定义 `java.lang.String` 覆盖核心类（Bootstrap 先加载）。
2. **唯一性**：同一个类在 JVM 中只有一份（避免重复加载）。
3. **一致性**：核心类库统一由 Bootstrap 加载。

## 4. 打破双亲委派

### 场景 1：SPI（Service Provider Interface）

```java
// 核心类 java.sql.DriverManager（由 Bootstrap 加载）需要加载厂商实现（在 classpath）
// Bootstrap 看不到 classpath → 用线程上下文类加载器
ServiceLoader<Driver> drivers = ServiceLoader.load(Driver.class);
```

`ServiceLoader` 使用 `Thread.currentThread().getContextClassLoader()`。

### 场景 2：Tomcat 的 Webapp ClassLoader

- 每个 webapp 独立类加载器，实现应用隔离。
- **先自己加载**（除核心类），再委派父加载器（违反双亲委派）。
- 目的：不同应用可以使用不同版本的同一依赖。

### 场景 3：OSGi / 模块化热部署

每个 Bundle 一个类加载器，支持动态安装/卸载。

### 自定义类加载器实现

```java
public class CustomClassLoader extends ClassLoader {
    private final Path baseDir;

    public CustomClassLoader(Path baseDir, ClassLoader parent) {
        super(parent);
        this.baseDir = baseDir;
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        try {
            Path file = baseDir.resolve(name.replace('.', '/') + ".class");
            byte[] bytes = Files.readAllBytes(file);
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

使用：

```java
ClassLoader cl = new CustomClassLoader(Path.of("/plugins"), getClass().getClassLoader());
Class<?> clazz = cl.loadClass("com.acme.Plugin");
Object instance = clazz.getDeclaredConstructor().newInstance();
```

### 加密类加载器（防反编译）

```java
protected Class<?> findClass(String name) throws ClassNotFoundException {
    byte[] encrypted = readEncrypted(name);
    byte[] bytes = decrypt(encrypted);            // AES 解密
    return defineClass(name, bytes, 0, bytes.length);
}
```

> 只是提高门槛，无法真正防破解。

## 5. 类的唯一性

```java
Class<?> c1 = cl1.loadClass("com.acme.User");
Class<?> c2 = cl2.loadClass("com.acme.User");
System.out.println(c1 == c2);      // false！不同加载器加载的同一个类是两个类
```

后果：

- `ClassCastException: com.acme.User cannot be cast to com.acme.User`（诡异报错）。
- `instanceof` 为 false。
- 静态变量不共享。

常见原因：同一个 jar 被不同类加载器加载（Tomcat 应用、OSGi、插件系统）。

## 6. 类的卸载

条件（**同时满足**）：

1. 该类的所有实例已被回收。
2. 加载该类的 `ClassLoader` 已被回收。
3. 该类的 `Class` 对象无引用（无反射、无 JNI、无静态引用）。

由 Bootstrap 加载的核心类**永不卸载**。

热部署泄漏的典型原因：自定义类加载器被线程/ThreadLocal/静态字段持有 → 无法卸载 → Metaspace OOM。

## 7. `Class.forName` vs `ClassLoader.loadClass`

```java
Class.forName("com.mysql.cj.jdbc.Driver");          // 初始化类（执行静态块）
Class.forName("X", false, loader);                   // 不初始化
loader.loadClass("com.acme.User");                   // 只加载，不初始化
```

JDBC 驱动注册依赖 `Class.forName` 触发静态块：

```java
static {
    DriverManager.registerDriver(new Driver());
}
```

## 8. 常见异常

| 异常 | 原因 |
|---|---|
| `ClassNotFoundException` | 显式加载时找不到类（classpath 缺 jar） |
| `NoClassDefFoundError` | 编译时存在，运行时找不到（依赖缺失/静态初始化失败） |
| `UnsupportedClassVersionError` | 编译版本高于 JVM |
| `ClassCastException` | 同名的不同类（不同加载器） |
| `LinkageError` | 类版本/依赖冲突（多版本同一类） |
| `ExceptionInInitializerError` | 静态初始化抛异常 |
| `IllegalAccessError` | 访问了不可见的类/方法 |

```java
// 排查 classpath
System.out.println(System.getProperty("java.class.path"));
// 或启动时 -verbose:class 查看加载来源
```

## 9. JDK 9+ 模块化影响

- Bootstrap 加载 `java.base` 等核心模块。
- Platform ClassLoader 加载部分标准模块。
- App ClassLoader 加载 classpath 与应用模块。
- 模块内类不可见其他模块未导出的包（强封装）。

## 10. 实战：插件系统

```java
public class PluginManager {
    private final Map<String, ClassLoader> loaders = new ConcurrentHashMap<>();

    public Plugin load(Path jar) throws Exception {
        URLClassLoader cl = new URLClassLoader(
                new URL[]{jar.toUri().toURL()},
                getClass().getClassLoader());
        Class<?> clazz = cl.loadClass("com.acme.PluginImpl");
        Plugin p = (Plugin) clazz.getDeclaredConstructor().newInstance();
        loaders.put(p.name(), cl);
        return p;
    }

    public void unload(String name) throws IOException {
        URLClassLoader cl = (URLClassLoader) loaders.remove(name);
        if (cl != null) cl.close();       // 还需确保无引用残留
    }
}
```

## 11. 常见坑

1. 以为静态变量在「加载」阶段就赋值（实际在初始化）。
2. 用 `Class.forName` 却期望不初始化。
3. 自定义类加载器忘记 `findClass` 用 `defineClass`。
4. 重写 `loadClass` 时破坏双亲委派导致核心类冲突。
5. 热部署后 Metaspace 持续增长（类加载器泄漏）。
6. 多模块同名类冲突（`NoSuchMethodError`）。
7. SPI 实现未放在 `META-INF/services/` 目录。
8. 依赖冲突时 `NoClassDefFoundError` 排查困难（用 `mvn dependency:tree`）。
9. `ClassCastException` 同名类（检查类加载器）。
10. 静态初始化块抛异常 → `ExceptionInInitializerError` 且类不可用。

## 12. 速查

| 概念 | 要点 |
|---|---|
| 加载 | 读字节码 → 方法区 → 堆中 Class 对象 |
| 准备 | 静态变量赋零值 |
| 初始化 | `<clinit>`，懒加载，线程安全 |
| 双亲委派 | 父优先，保证安全与唯一 |
| 打破委派 | SPI（上下文加载器）、Tomcat、OSGi |
| 类唯一性 | 类加载器 + 全限定名 |
| 卸载 | 无实例、无 Class 引用、加载器可回收 |
| 调试 | `-verbose:class`、`jmap -clstats` |
