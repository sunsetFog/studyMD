# 51 · 反射

## 1. 什么是反射

运行时动态获取类信息并操作对象（创建实例、调用方法、访问字段）。

```java
Class<?> clazz = Class.forName("com.acme.User");
Object obj = clazz.getDeclaredConstructor().newInstance();
Method m = clazz.getMethod("getName");
Object name = m.invoke(obj);
```

用途：框架（Spring、MyBatis、Jackson）、注解处理、动态代理、序列化。

## 2. 获取 Class 对象的四种方式

```java
Class<?> c1 = String.class;                    // 类字面量（最快，不触发初始化）
Class<?> c2 = "abc".getClass();                // 实例方法
Class<?> c3 = Class.forName("java.lang.String"); // 触发初始化
Class<?> c4 = ClassLoader.getSystemClassLoader().loadClass("java.lang.String"); // 不初始化

// 基本类型
int.class;                    // int
Integer.TYPE;                 // int
Integer.class;                // java.lang.Integer
int.class == Integer.TYPE;    // true

// 数组
String[].class;
int[].class;
```

## 3. 类信息

```java
Class<?> c = User.class;

c.getName();                   // com.acme.User
c.getSimpleName();             // User
c.getPackageName();            // com.acme
c.getModifiers();              // 修饰符位
Modifier.toString(c.getModifiers());
c.getSuperclass();             // 父类
c.getInterfaces();             // 接口
c.isInterface(); c.isEnum(); c.isRecord(); c.isAnnotation();
c.isArray(); c.isPrimitive();
c.getTypeName();

c.getDeclaredFields();         // 本类字段（含 private）
c.getFields();                 // public 字段（含继承）
c.getDeclaredMethods();
c.getMethods();                // public 方法（含继承）
c.getDeclaredConstructors();
c.getConstructors();
c.getAnnotations();
```

## 4. 创建实例

```java
// 1. 无参构造
User u = User.class.getDeclaredConstructor().newInstance();

// 2. 有参构造
Constructor<User> ctor = User.class.getDeclaredConstructor(Long.class, String.class);
User u2 = ctor.newInstance(1L, "Tom");

// 3. 访问私有构造器
Constructor<User> priv = User.class.getDeclaredConstructor(String.class);
priv.setAccessible(true);
User u3 = priv.newInstance("secret");

// 4. 已废弃：Class.newInstance()（异常处理不友好，Java 9 deprecated）
```

## 5. 操作字段

```java
Class<?> c = User.class;
Object u = c.getDeclaredConstructor().newInstance();

Field f = c.getDeclaredField("name");
f.setAccessible(true);                // 私有字段必须
f.set(u, "Tom");
Object v = f.get(u);

// 静态字段
Field s = c.getDeclaredField("count");
s.setAccessible(true);
s.set(null, 100);                     // 实例传 null

// 基本类型用 setInt/getInt 等避免装箱
Field age = c.getDeclaredField("age");
age.setAccessible(true);
age.setInt(u, 18);
int a = age.getInt(u);
```

### setAccessible 与模块系统

Java 9+ 强封装：反射访问其他模块的非 public 成员会抛 `InaccessibleObjectException`。

```bash
--add-opens java.base/java.lang=ALL-UNNAMED
```

```java
// 或代码中
Module mod = User.class.getModule();
mod.isOpen("com.acme");
```

## 6. 调用方法

```java
Method m = c.getDeclaredMethod("greeting", String.class);
m.setAccessible(true);
Object result = m.invoke(u, "Tom");       // 静态方法第一个参数传 null

// 可变参数
Method var = c.getMethod("log", String.class, Object[].class);
var.invoke(u, "fmt", new Object[]{1, 2});

// 异常：invoke 抛 InvocationTargetException，真实异常在 getCause()
try {
    m.invoke(u);
} catch (InvocationTargetException e) {
    Throwable real = e.getTargetException();
}
```

## 7. 泛型信息

```java
public class Repo<T> {
    private List<User> users;
    public Map<String, List<User>> map;
    public <R> R transform(List<R> input) { return null; }
}

// 字段泛型
Field f = Repo.class.getDeclaredField("users");
ParameterizedType pt = (ParameterizedType) f.getGenericType();
Class<?> raw = (Class<?>) pt.getRawType();                    // List
Type arg = pt.getActualTypeArguments()[0];                    // User

// 方法泛型
Method m = Repo.class.getMethod("transform", List.class);
Type[] params = m.getGenericParameterTypes();
Type ret = m.getGenericReturnType();

// 类型处理（可能是 Class / ParameterizedType / TypeVariable / WildcardType / GenericArrayType）
```

## 8. 反射性能

| 方式 | 相对耗时 |
|---|---|
| 直接调用 | 1x |
| `Method.invoke` | 3-20x |
| `setAccessible(true)` 后 | 稍快 |
| `MethodHandle` | 2-5x |
| LambdaMetafactory | 接近直接调用 |

优化手段：

```java
// 1. 缓存 Method/Field/Constructor
private static final Method GET_NAME;
static {
    try { GET_NAME = User.class.getMethod("getName"); }
    catch (NoSuchMethodException e) { throw new ExceptionInInitializerError(e); }
}

// 2. setAccessible(true) 跳过访问检查
method.setAccessible(true);

// 3. MethodHandle（Java 7+）
MethodHandles.Lookup lookup = MethodHandles.lookup();
MethodHandle mh = lookup.findVirtual(User.class, "getName", MethodType.methodType(String.class));
String name = (String) mh.invokeExact(user);

// 4. VarHandle（Java 9+，替代 Unsafe 字段操作）
VarHandle vh = MethodHandles.lookup().findVarHandle(User.class, "name", String.class);
String v = (String) vh.get(user);
```

> JIT 对内联友好的反射（热点代码）会做优化，但复杂场景仍需缓存。

## 9. 实战：通用 toString

```java
public static String toString(Object obj) {
    if (obj == null) return "null";
    Class<?> c = obj.getClass();
    StringBuilder sb = new StringBuilder(c.getSimpleName()).append('{');
    boolean first = true;
    for (Field f : c.getDeclaredFields()) {
        if (Modifier.isStatic(f.getModifiers())) continue;
        f.setAccessible(true);
        if (!first) sb.append(", ");
        first = false;
        try {
            sb.append(f.getName()).append('=').append(f.get(obj));
        } catch (IllegalAccessException e) {
            sb.append(f.getName()).append("=?");
        }
    }
    return sb.append('}').toString();
}
```

## 10. 实战：简易 Bean 拷贝

```java
public static <T> T copy(Object src, Class<T> targetType) throws Exception {
    T target = targetType.getDeclaredConstructor().newInstance();
    Map<String, Field> srcFields = new HashMap<>();
    for (Field f : src.getClass().getDeclaredFields()) srcFields.put(f.getName(), f);

    for (Field tf : targetType.getDeclaredFields()) {
        Field sf = srcFields.get(tf.getName());
        if (sf == null || !sf.getType().equals(tf.getType())) continue;
        sf.setAccessible(true);
        tf.setAccessible(true);
        tf.set(target, sf.get(src));
    }
    return target;
}
```

> 生产用 MapStruct（编译期生成，零反射）或 BeanUtils（注意性能）。

## 11. 反射的局限

1. 性能开销（可缓存缓解）。
2. 破坏封装（可访问私有成员）。
3. 编译期无法检查（拼写错误运行时才暴露）。
4. 模块系统限制（需 `--add-opens`）。
5. 无法优化（JIT 难内联，尤其动态生成的调用）。
6. 安全性（可绕过访问控制）。
7. 代码可读性差、IDE 支持弱。

## 12. 常见异常

| 异常 | 原因 |
|---|---|
| `ClassNotFoundException` | 类不存在/类名错 |
| `NoSuchMethodException` | 方法签名不匹配 |
| `NoSuchFieldException` | 字段不存在 |
| `IllegalAccessException` | 未 setAccessible 或权限不足 |
| `InstantiationException` | 抽象类/接口/无无参构造 |
| `InvocationTargetException` | 被调用方法内部抛异常 |
| `InaccessibleObjectException` | 模块未开放（JDK 9+） |
| `IllegalArgumentException` | 参数类型不匹配 |

## 13. 常见坑

1. `getMethod` 只找 public（含继承），`getDeclaredMethod` 找本类所有（不含继承）。
2. 参数类型必须精确匹配（`int` vs `Integer`）。
3. 未 `setAccessible(true)` 访问私有成员。
4. 把 `InvocationTargetException` 当业务异常打印（应取 cause）。
5. 每次调用都 `getMethod`（未缓存）。
6. 反射创建对象要求无参构造器（否则需 `getDeclaredConstructor(参数)`）。
7. 泛型信息在字段/方法签名上，运行时集合本身无泛型。
8. 修改 final 字段（需额外手段，通常不该做）。
9. 反射绕过泛型检查（`List<String>` 可塞 Integer）。
10. 在 GraalVM Native Image 中反射需注册配置。

## 14. 速查

| 操作 | API |
|---|---|
| 获取 Class | `X.class` / `getClass()` / `Class.forName()` |
| 创建实例 | `getDeclaredConstructor(...).newInstance(...)` |
| 字段 | `getDeclaredField` + `setAccessible` + `get/set` |
| 方法 | `getDeclaredMethod` + `invoke` |
| 构造器 | `getDeclaredConstructor` |
| 注解 | `getAnnotation(X.class)` |
| 泛型 | `getGenericType` / `ParameterizedType` |
| 高性能 | `MethodHandle` / `VarHandle` |
| 缓存 | `static final Map<String, Method>` |
