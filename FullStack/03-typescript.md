# 03 - TypeScript

## 1. TS 为什么存在

编译期类型安全 + 更好的 IDE 重构体验 + 类型即文档。核心 trade-off：开发期成本 vs 大型项目可维护性。

## 2. type vs interface（必考）

| 维度 | interface | type |
|---|---|---|
| 对象类型 | 适合 | 可以 |
| 声明合并 | 支持（同名自动合并） | 不支持（重名报错） |
| extends | 只能 extends 对象/接口 | 可用 & 组合任意类型 |
| 联合/映射/条件类型 | 不行 | 行 |

结论：对外公开的 API 形状用 interface（可被扩展），工具类型/联合用 type。

## 3. 泛型（必考）

```ts
function first<T>(arr: T[]): T | undefined { return arr[0]; }

// 泛型约束
function getLen<T extends { length: number }>(x: T) { return x.length; }

// 默认泛型 & keyof
function pick<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }
```

`keyof`/`typeof`/`in`/`as` 组合是高级题来源。

## 4. 常用工具类型（背）

```ts
Partial<T>        // 全部可选
Required<T>       // 全部必填
Readonly<T>       // 只读
Pick<T, K>        // 挑选
Omit<T, K>        // 排除
Record<K, V>      // 键值映射
ReturnType<F>     // 函数返回类型
Parameters<F>
Awaited<T>        // 解包 Promise
NonNullable<T>
Exclude<T, U> / Extract<T, U>
```

手写工具类型是中级面试高频：`MyPartial`、`MyPick`、`DeepPartial`。

```ts
type MyPick<T, K extends keyof T> = { [P in K]: T[P] };
type DeepPartial<T> = T extends object ? { [P in keyof T]?: DeepPartial<T[P]> } : T;
```

## 5. 高级类型技巧

```ts
// 联合类型 & 可辨识联合（前端状态建模核心）
type State =
  | { status: 'loading' }
  | { status: 'success'; data: User[] }
  | { status: 'error'; msg: string };
// switch(state.status) 获得类型收窄

// 条件类型 + infer
type UnPromisify<T> = T extends Promise<infer U> ? U : T;

// 模板字面量类型（4.1+）
type Route = `/${string}`;
```

- 类型收窄手段：typeof/in/instanceof/判别字段/断言函数
- `unknown` vs `any`：unknown 安全（需收窄后使用），any 关闭检查
- `as const`：字面量只读推断，常用于 action 常量

## 6. 工程实践考点

- `tsconfig` 关键项：`strict: true`（含 strictNullChecks）、`noUncheckedIndexedAccess`
- 声明文件 `.d.ts`、模块增强（给第三方库补类型）
- 类型体操适度原则：业务代码以可读性优先
- Vue3 全量 TS 实现 / React+TSX：`defineProps<T>()`、泛型组件、`forwardRef<T, P>`

## 常见追问

1. 枚举 vs 字面量联合：enum 生成运行时代码（const enum 例外），字面量联合零开销——现代更推荐后者
2. `void` vs `undefined`：void 表示"不关心返回值"
3. 泛型在编译后会怎样？——擦除，运行时无类型信息

## Checklist

- [ ] type/interface 选型能说清依据
- [ ] 会手写 Pick/Omit/DeepPartial
- [ ] 用可辨识联合建模 API 状态
