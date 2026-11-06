# 54 · Class 文件与字节码

## 1. Class 文件结构

```
ClassFile {
    u4 magic;                 // 0xCAFEBABE
    u2 minor_version;
    u2 major_version;         // 52=Java8, 61=Java17, 65=Java21
    u2 constant_pool_count;
    cp_info constant_pool[];  // 常量池
    u2 access_flags;
    u2 this_class;
    u2 super_class;
    u2 interfaces_count;
    u2 interfaces[];
    u2 fields_count;
    field_info fields[];
    u2 methods_count;
    method_info methods[];
    u2 attributes_count;
    attribute_info attributes[];
}
```

### 版本号对照

| Java | major |
|---|---|
| 8 | 52 |
| 11 | 55 |
| 17 | 61 |
| 21 | 65 |
| 25 | 69 |

### 常量池类型

| tag | 类型 |
|---|---|
| 1 | Utf8 |
| 3 | Integer |
| 4 | Float |
| 5 | Long |
| 6 | Double |
| 7 | Class |
| 8 | String |
| 9 | Fieldref |
| 10 | Methodref |
| 11 | InterfaceMethodref |
| 12 | NameAndType |
| 15 | MethodHandle |
| 16 | MethodType |
| 18 | InvokeDynamic |

```bash
javap -v Hello.class        # 查看常量池
```

## 2. 字节码指令

### 加载与存储

| 指令 | 含义 |
|---|---|
| `iload_<n>` | 加载 int 局部变量到栈 |
| `aload_<n>` | 加载引用 |
| `istore_<n>` | 存入局部变量 |
| `ldc` | 加载常量池常量 |
| `bipush` / `sipush` | 加载 byte/short 常量 |

### 运算

`iadd` `isub` `imul` `idiv` `irem` `ineg` `ishl` `ishr` `iushr` `iand` `ior` `ixor`

### 方法调用

| 指令 | 用途 |
|---|---|
| `invokestatic` | 静态方法 |
| `invokespecial` | 构造器、private、super |
| `invokevirtual` | 实例方法（虚分派） |
| `invokeinterface` | 接口方法 |
| `invokedynamic` | 动态调用（Lambda、字符串拼接） |

### 返回

`ireturn` `lreturn` `freturn` `dreturn` `areturn` `return`

### 其他

`new` `newarray` `anewarray` `checkcast` `instanceof` `monitorenter` `monitorexit` `athrow` `if_icmpgt` `goto` `tableswitch` `lookupswitch`

## 3. 示例：反编译

```java
public class Demo {
    public int add(int a, int b) {
        return a + b;
    }

    public String greet(String name) {
        return "Hello, " + name;
    }
}
```

```bash
javac Demo.java && javap -c Demo
```

```
public int add(int, int);
  Code:
     0: iload_1
     1: iload_2
     2: iadd
     3: ireturn

public java.lang.String greet(java.lang.String);
  Code:
     0: aload_1
     1: invokedynamic #7,  0  // InvokeDynamic #0:makeConcatWithConstants
     6: areturn
```

Java 9+ 字符串拼接用 `invokedynamic` + `StringConcatFactory`，比 StringBuilder 更灵活。

## 4. 栈帧与局部变量表

```java
public static int calc(int a) {
    int b = a * 2;
    return b + 1;
}
```

| slot | 内容 |
|---|---|
| 0 | 参数 a |
| 1 | 局部变量 b |

实例方法 slot 0 是 `this`（静态方法没有）。

```bash
javap -v -p Demo.class     # 看 LocalVariableTable
```

## 5. 异常表

```java
void f() {
    try { g(); }
    catch (IOException e) { h(); }
}
```

编译为 `Exception table`：

```
from  to  target  type
  0    5    8     Class java/io/IOException
```

finally 通过复制代码或 `athrow` 实现（Java 7+ 用 `jsr` 的子程序方式已废弃）。

## 6. 字节码增强技术

| 技术 | 层次 | 特点 |
|---|---|---|
| **ASM** | 字节码指令 | 最快、最底层、难用 |
| **Javassist** | 源码字符串 | 简单，性能一般 |
| **ByteBuddy** | 流式 API | 现代首选，类型安全 |
| **CGLIB** | 基于 ASM | 代理专用 |
| **Java Agent** | 类加载时改写 | 无侵入（SkyWalking、Arthas） |

### ASM 示例：生成类

```java
ClassWriter cw = new ClassWriter(0);
cw.visit(V17, ACC_PUBLIC, "com/acme/Generated", null, "java/lang/Object", null);

// 默认构造器
MethodVisitor mv = cw.visitMethod(ACC_PUBLIC, "<init>", "()V", null, null);
mv.visitCode();
mv.visitVarInsn(ALOAD, 0);
mv.visitMethodInsn(INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
mv.visitInsn(RETURN);
mv.visitMaxs(1, 1);
mv.visitEnd();

cw.visitEnd();
byte[] bytes = cw.toByteArray();
```

### ByteBuddy 示例

```java
Class<?> dynamic = new ByteBuddy()
        .subclass(Object.class)
        .name("com.acme.Generated")
        .defineMethod("hello", String.class, Visibility.PUBLIC)
        .intercept(FixedValue.value("hi"))
        .make()
        .load(ByteBuddyExample.class.getClassLoader())
        .getLoaded();

Object o = dynamic.getDeclaredConstructor().newInstance();
System.out.println(dynamic.getMethod("hello").invoke(o));   // hi
```

### Javassist 示例

```java
ClassPool pool = ClassPool.getDefault();
CtClass cc = pool.get("com.acme.UserService");

CtMethod m = cc.getDeclaredMethod("get");
m.insertBefore("long start = System.nanoTime();");
m.insertAfter("System.out.println(\"cost \" + (System.nanoTime() - start));");

Class<?> clazz = cc.toClass();
```

## 7. Java Agent（无侵入增强）

```java
public class TimingAgent {
    public static void premain(String args, Instrumentation inst) {
        inst.addTransformer(new ClassFileTransformer() {
            @Override
            public byte[] transform(ClassLoader loader, String className,
                                    Class<?> classBeingRedefined,
                                    ProtectionDomain pd, byte[] bytes) {
                if (!className.startsWith("com/acme/")) return null;
                // 用 ASM/ByteBuddy 改写字节码
                return enhance(bytes);
            }
        });
    }
}
```

```
META-INF/MANIFEST.MF:
Premain-Class: com.acme.TimingAgent
```

```bash
java -javaagent:agent.jar -jar app.jar
```

应用：APM（SkyWalking、Pinpoint）、Arthas、热部署（JRebel）、Mock 工具。

## 8. invokedynamic 与 Lambda

```java
Runnable r = () -> System.out.println("hi");
```

编译后：

```
invokedynamic #0  // InvokeDynamic #0:run:()Ljava/lang/Runnable;
```

`invokedynamic` 首次执行时调用 `LambdaMetafactory` 生成一个实现 Runnable 的类（或复用），后续直接调用。这比匿名内部类更省类文件，且 JIT 后可内联。

## 9. 字节码查看工具

| 工具 | 说明 |
|---|---|
| `javap` | JDK 自带反汇编 |
| **ASM Bytecode Viewer** | IDEA 插件，可视化 |
| **JClassLib** | IDEA 插件，结构化查看 |
| **jbe** | 独立 GUI |
| **Bytecode Viewer** | 反编译集合 |

```bash
javap -c -p -v -s MyClass.class
```

## 10. 运行时字节码操作的应用

| 场景 | 技术 |
|---|---|
| AOP 代理 | CGLIB / ByteBuddy |
| Mock 测试 | ByteBuddy（Mockito 5） |
| ORM 延迟加载 | Javassist（Hibernate） |
| APM 埋点 | Java Agent + ASM |
| 热部署 | JRebel / Spring Boot DevTools |
| 代码生成 | ASM（FastJSON、Dubbo） |
| 加密保护 | 自定义类加载器 + 解密 |

## 11. 常见坑

1. 修改字节码后未更新 `StackMapTable`（Java 7+ 校验失败）。
2. 常量池索引错误（`ClassFormatError`）。
3. 忘记 `visitMaxs`（ASM）。
4. 增强的类被 Bootstrap 加载的类引用（类加载器不一致）。
5. Java Agent 中 `transform` 返回 null 表示不修改（不是空字节数组）。
6. 重定义类（retransform）不能改方法签名/字段。
7. Lambda 的 `invokedynamic` 在低版本 JVM 上不兼容。
8. 字节码版本与 JVM 不匹配（`UnsupportedClassVersionError`）。
9. 增强后调试信息丢失（行号表）。
10. 忘记处理 `module-info` 与 `nestmates`。

## 12. 速查

| 需求 | 命令/工具 |
|---|---|
| 看字节码 | `javap -c -p` |
| 看常量池 | `javap -v` |
| 生成类 | ASM / ByteBuddy |
| 修改已有类 | Java Agent + ASM |
| 可视化 | JClassLib / ASM Viewer |
| 类版本 | major 52=8, 61=17, 65=21 |
| 魔数 | `0xCAFEBABE` |
| 方法调用 | `invokevirtual` / `invokeinterface` / `invokedynamic` |
