# TypeScript 知识点总结

## 1. 简介

TypeScript 是 JavaScript 的超集，在 JS 基础上增加了静态类型系统，编译后输出纯 JavaScript。

特点：
- 静态类型检查，编译期发现错误
- 完全兼容 JavaScript
- 支持最新的 ES 特性
- 更好的 IDE 提示和重构支持

## 2. 安装与编译

```
# 全局安装
npm install -g typescript

# 编译单个文件
tsc index.ts

# 生成配置文件
tsc --init

# 监听编译
tsc -w
```

## 3. 基础类型

```
// 显式声明类型
let name: string = 'tom'
let age: number = 20
let isDone: boolean = true

// 类型推导：不写类型也会自动推导
let count = 10          // 推导为 number
// count = 'str'        // 报错

// 数组
let list: number[] = [1, 2, 3]
let list2: Array<string> = ['a', 'b']

// null 和 undefined
let u: undefined = undefined
let n: null = null
```

## 4. 元组 Tuple

固定长度和类型的数组。

```
let tuple: [string, number] = ['tom', 20]

tuple[0]            // 'tom'，类型 string
tuple[1]            // 20，类型 number
// tuple = [20, 'tom']   // 报错，类型顺序不对

// 可选元素
let opt: [string, number?] = ['tom']
```

## 5. 联合类型与交叉类型

```
// 联合类型：值可以是多种类型之一（或）
let id: string | number
id = 'abc'
id = 123

// 交叉类型：同时满足多个类型（与）
interface A { name: string }
interface B { age: number }
type C = A & B
const c: C = { name: 'tom', age: 20 }   // 必须同时有 name 和 age
```

## 6. 类型别名 type

给类型起一个名字，便于复用。

```
type ID = string | number
type Point = { x: number; y: number }
type Callback = (data: string) => void

const id: ID = 123
const p: Point = { x: 1, y: 2 }
const fn: Callback = (data) => console.log(data)
```

## 7. 接口 interface

定义对象的结构，描述形状。

```
interface User {
    name: string
    age: number
    email?: string          // 可选属性
    readonly id: number     // 只读属性
    sayHi?: () => void      // 方法
}

const user: User = {
    name: 'tom',
    age: 20,
    id: 1
}
// user.id = 2   // 报错，只读属性不可改

// 接口继承
interface Admin extends User {
    role: string
}

// 函数接口
interface AddFn {
    (a: number, b: number): number
}
```

## 8. type 与 interface 的区别

| | type | interface |
|--|------|-----------|
| 扩展 | 用 `&` 交叉 | 用 `extends` |
| 声明合并 | 不支持 | 支持（同名自动合并） |
| 适用 | 联合、元组、工具类型 | 对象结构 |

```
// interface 声明合并
interface User {
    name: string
}
interface User {
    age: number
}
// 最终 User 同时有 name 和 age

// type 不能重复声明
type Point = { x: number }
// type Point = { y: number }   // 报错
```

## 9. 函数类型

```
// 参数和返回值类型
function add(a: number, b: number): number {
    return a + b
}

// 函数表达式
const mul: (a: number, b: number) => number = (a, b) => a * b

// 可选参数（必须在必选参数之后）
function greet(name: string, age?: number) {
    return age ? `${name}:${age}` : name
}

// 默认参数
function hello(name: string = 'world') {
    return `Hello ${name}`
}

// 剩余参数
function sum(...nums: number[]): number {
    return nums.reduce((s, n) => s + n, 0)
}

// 函数重载
function pick(x: string): string
function pick(x: number): number
function pick(x: string | number) {
    return x
}
```

## 10. 泛型

类型参数化，编写可复用的组件。

```
// 泛型函数
function identity<T>(value: T): T {
    return value
}
identity<string>('hello')   // 显式指定
identity(123)               // 自动推导为 number

// 泛型接口
interface Result<T> {
    code: number
    data: T
}
const r: Result<User> = { code: 200, data: { name: 'tom', age: 20 } }

// 泛型类
class Box<T> {
    constructor(public value: T) {}
}
const box = new Box<string>('hello')

// 多个泛型参数
function pair<T, U>(a: T, b: U): [T, U] {
    return [a, b]
}
```

## 11. 泛型约束

用 extends 限制泛型的范围。

```
// 约束 T 必须有 length 属性
function getLength<T extends { length: number }>(arg: T): number {
    return arg.length
}
getLength('hello')   // 5
getLength([1, 2])    // 2
// getLength(123)    // 报错，number 没有 length

// 使用 keyof 约束键
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key]
}
getProp({ name: 'tom' }, 'name')   // 'tom'
```

## 12. 类型断言

告诉编译器「我知道这个值的类型」。

```
// as 语法（推荐）
const el = document.getElementById('app') as HTMLDivElement

// 尖括号语法（JSX 中不能用）
const num = <number>someValue

// 双重断言（少用）
const x = value as unknown as string
```

## 13. 枚举 enum

定义一组命名常量。

```
// 数字枚举（默认从 0 开始）
enum Color {
    Red,        // 0
    Green,      // 1
    Blue        // 2
}
Color.Red       // 0
Color[0]        // 'Red'

// 手动指定值
enum Status {
    Success = 200,
    Error = 500
}

// 字符串枚举
enum Direction {
    Up = 'UP',
    Down = 'DOWN'
}
```

## 14. 类 class

```
class Person {
    name: string
    age: number

    constructor(name: string, age: number) {
        this.name = name
        this.age = age
    }

    sayHi(): void {
        console.log(`Hi, ${this.name}`)
    }
}

const p = new Person('tom', 20)
p.sayHi()
```

## 15. 访问修饰符

```
class Person {
    public name: string       // 公开（默认）
    private age: number       // 私有，仅类内部可访问
    protected gender: string  // 受保护，子类也可访问

    constructor(name: string, age: number) {
        this.name = name
        this.age = age
    }
}

// 参数属性：构造器参数直接声明为属性（简写）
class User {
    constructor(public name: string, private age: number) {}
    // 等价于上面手动声明 + 赋值
}
```

## 16. 抽象类

不能被实例化，只能被继承，可定义抽象方法。

```
abstract class Animal {
    abstract makeSound(): void   // 抽象方法，子类必须实现

    move(): void {               // 普通方法
        console.log('moving')
    }
}

class Dog extends Animal {
    makeSound(): void {
        console.log('旺旺')
    }
}
```

## 17. 类型守卫

在代码块中缩小类型范围。

```
// typeof 类型守卫
function print(x: string | number) {
    if (typeof x === 'string') {
        console.log(x.toUpperCase())   // 这里 x 是 string
    } else {
        console.log(x.toFixed(2))      // 这里 x 是 number
    }
}

// instanceof 类型守卫
function getDate(x: Date | string) {
    if (x instanceof Date) {
        return x.getTime()             // x 是 Date
    }
    return x
}

// 自定义类型守卫
interface Cat { meow(): void }
interface Dog { bark(): void }

function isCat(animal: Cat | Dog): animal is Cat {
    return (animal as Cat).meow !== undefined
}
```

## 18. 字面量类型

把具体的值作为类型。

```
// 字符串字面量类型
type Direction = 'up' | 'down' | 'left' | 'right'
let dir: Direction = 'up'
// dir = 'abc'   // 报错

// 数字字面量类型
type Dice = 1 | 2 | 3 | 4 | 5 | 6
let dice: Dice = 6

// 结合使用
function request(method: 'GET' | 'POST', url: string) {}
request('GET', '/api')
```

## 19. keyof 与 typeof

```
// keyof：获取类型的键组成的联合类型
interface User {
    name: string
    age: number
}
type UserKey = keyof User   // 'name' | 'age'

// typeof：获取变量的类型
const obj = { a: 1, b: 'x' }
type ObjType = typeof obj   // { a: number, b: string }

// 结合使用
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key]
}
```

## 20. 索引签名

定义对象的键值类型。

```
// 字符串索引签名
interface Dict {
    [key: string]: string
}
const d: Dict = { a: 'x', b: 'y' }

// 数字索引签名
interface StringArray {
    [index: number]: string
}
const arr: StringArray = ['a', 'b']
```

## 21. 工具类型

TS 内置的实用类型。

```
interface User {
    id: number
    name: string
    age: number
    email?: string
}

// Partial：所有属性可选
type PartialUser = Partial<User>

// Required：所有属性必填
type RequiredUser = Required<User>

// Readonly：所有属性只读
type ReadonlyUser = Readonly<User>

// Pick：挑选部分属性
type NameAge = Pick<User, 'name' | 'age'>   // { name, age }

// Omit：排除部分属性
type NoId = Omit<User, 'id'>                // { name, age, email? }

// Record：构造键值对类型
type Map1 = Record<string, number>          // { [k: string]: number }

// ReturnType：获取函数返回值类型
function fn() { return 123 }
type R = ReturnType<typeof fn>              // number
```

## 22. unknown 与 any

- `any`：关闭类型检查，不安全
- `unknown`：类型安全的任意值，使用前必须缩小类型

```
let a: any = 'hello'
a.toFixed()          // any 不报错，运行时可能崩溃

let u: unknown = 'hello'
// u.toFixed()      // 报错，unknown 不能直接调用
if (typeof u === 'string') {
    u.toUpperCase()  // 缩小类型后可用
}
```

## 23. never 类型

表示永远不存在的值的类型。

```
// 抛出异常的函数返回 never
function error(msg: string): never {
    throw new Error(msg)
}

// 死循环函数
function loop(): never {
    while (true) {}
}

// 穷尽检查
type Shape = 'circle' | 'square'
function area(s: Shape) {
    switch (s) {
        case 'circle':
            return 1
        case 'square':
            return 2
        default:
            const check: never = s   // 若漏了分支会编译报错
            return check
    }
}
```

## 24. readonly 与 as const

```
// readonly 只读属性
interface Point {
    readonly x: number
    readonly y: number
}
const p: Point = { x: 1, y: 2 }
// p.x = 3   // 报错

// readonly 数组
const arr: readonly number[] = [1, 2, 3]
// arr.push(4)   // 报错

// as const：断言为字面量类型
const colors = ['red', 'green'] as const
// colors 类型为 readonly ['red', 'green']
```

## 25. 可选链与空值合并

```
const user = {
    info: {
        name: 'tom'
    }
}

// 可选链
user.info?.name        // 'tom'
user.address?.city     // undefined，不报错

// 空值合并
const name = user.info.name ?? '默认名'   // 'tom'
const val = null ?? 10                    // 10
```

## 26. 模块与命名空间

```
// 导出 export
// math.ts
export const add = (a: number, b: number) => a + b
export default function mul(a: number, b: number) {
    return a * b
}

// 导入 import
// main.ts
import mul, { add } from './math'
add(1, 2)

// 命名空间（较少用）
namespace Utils {
    export function log(msg: string) {
        console.log(msg)
    }
}
Utils.log('hello')
```

## 27. 类型声明文件 .d.ts

为 JavaScript 库提供类型定义。

```
// types.d.ts
declare module 'my-lib' {
    export function doSomething(x: string): number
}

// 全局变量声明
declare const VERSION: string

// 全局函数声明
declare function globalFn(x: number): void

// 声明全局接口
interface Window {
    myCustomProp: string
}
```

## 28. tsconfig.json 配置

```
{
  "compilerOptions": {
    "target": "ES2020",              // 编译目标版本
    "module": "ESNext",              // 模块规范
    "strict": true,                  // 严格模式（推荐开启）
    "outDir": "./dist",              // 输出目录
    "rootDir": "./src",              // 源码目录
    "esModuleInterop": true,         // 兼容 CommonJS 导入
    "skipLibCheck": true,            // 跳过声明文件检查
    "noImplicitAny": true,           // 禁止隐式 any
    "strictNullChecks": true,        // 严格空值检查
    "paths": {                       // 路径别名
      "@/*": ["src/*"]
    }
  },
  "include": ["src"],
  "exclude": ["node_modules", "dist"]
}
```

## 29. 泛型与接口组合实战

```
// 泛型约束 + 工具类型的组合
interface ApiResponse<T> {
    code: number
    message: string
    data: T
}

interface User {
    id: number
    name: string
}

// 请求函数返回泛型类型
async function request<T>(url: string): Promise<ApiResponse<T>> {
    const res = await fetch(url)
    return res.json()
}

// 使用时自动推导类型
const result = await request<User>('/api/user')
result.data.id     // number
result.data.name   // string
```

## 30. 常用类型速查

| 类型 | 说明 | 示例 |
|------|------|------|
| string | 字符串 | 'hello' |
| number | 数字 | 123 |
| boolean | 布尔 | true |
| any | 任意类型（慎用） | - |
| unknown | 未知类型（安全） | - |
| never | 永不返回 | - |
| void | 无返回值 | - |
| null / undefined | 空值 | - |
| T[] | 数组 | number[] |
| [T, U] | 元组 | [string, number] |
| T \| U | 联合类型 | string \| number |
| T & U | 交叉类型 | A & B |
| Partial<T> | 全部可选 | - |
| Pick<T, K> | 挑选属性 | - |
| Record<K, V> | 键值对 | - |
