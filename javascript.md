# JavaScript 知识点总结

## 1. 简介

JavaScript 是一门解释型、动态类型、基于原型的编程语言，用于 Web 交互，可在浏览器和 Node.js 中运行。

特点：
- 单线程、事件驱动
- 弱类型、动态类型
- 基于原型继承
- 支持函数式与面向对象编程

## 2. 变量声明 var / let / const

| 关键字 | 作用域 | 能否重复声明 | 能否重新赋值 | 变量提升 |
|--------|--------|--------------|--------------|----------|
| var | 函数作用域 | 能 | 能 | 有 |
| let | 块级作用域 | 否 | 能 | 否（暂时性死区）|
| const | 块级作用域 | 否 | 否 | 否 |

```
var a = 1
let b = 2
const c = 3

// const 定义的对象属性可改
const obj = { name: 'tom' }
obj.name = 'jerry'   // 合法
// obj = {}          // 报错，不能重新赋值
```

## 3. 数据类型

基本类型：String、Number、Boolean、Null、Undefined、Symbol、BigInt。
引用类型：Object、Array、Function、Date 等。

```
typeof 'hello'        // "string"
typeof 1              // "number"
typeof true           // "boolean"
typeof undefined      // "undefined"
typeof null           // "object"（历史遗留）
typeof Symbol()       // "symbol"
typeof 10n            // "bigint"
typeof {}             // "object"
typeof []             // "object"
typeof function(){}   // "function"
```

## 4. 类型转换

JS 在运算中会自动做类型转换，理解隐式转换规则能避免常见的比较错误。

```
// 显式转换
Number('123')      // 123
String(123)        // "123"
Boolean(0)         // false

// 隐式转换
'5' + 1            // "51"（字符串拼接）
'5' - 1            // 4
'5' * 2            // 10
1 == '1'           // true（宽松相等，会转换类型）
1 === '1'          // false（严格相等，推荐）
```

## 5. 字符串常用方法

字符串是不可变的，以下方法都返回新字符串，不会修改原字符串。

```
const s = 'Hello World'

s.length            // 11
s.toUpperCase()     // "HELLO WORLD"
s.toLowerCase()     // "hello world"
s.includes('World') // true
s.indexOf('o')      // 4
s.slice(0, 5)       // "Hello"
s.split(' ')        // ["Hello", "World"]
s.replace('World', 'JS')  // "Hello JS"
s.trim()            // 去除首尾空格
```

## 6. 数组常用方法

数组是可变对象，其中 push/pop/shift/unshift/splice 会原地修改原数组。

```
const arr = [1, 2, 3, 4, 5]

arr.push(6)          // 尾部添加，返回长度
arr.pop()            // 尾部删除，返回元素
arr.shift()          // 头部删除
arr.unshift(0)       // 头部添加
arr.slice(1, 3)      // [2, 3] 截取
arr.splice(1, 2)     // 删除索引 1 起 2 个元素
arr.includes(3)      // true
arr.indexOf(3)       // 2
arr.join('-')        // "1-2-3-4-5"
arr.concat([6, 7])   // 合并
```

## 7. 数组遍历方法

这些高阶方法接收回调函数，是函数式编程的核心，注意各方法的返回值差异。

```
const arr = [1, 2, 3, 4]

// map：返回新数组
arr.map(x => x * 2)            // [2, 4, 6, 8]

// filter：过滤
arr.filter(x => x > 2)         // [3, 4]

// reduce：累加
arr.reduce((sum, x) => sum + x, 0)  // 10

// forEach：遍历（无返回值）
arr.forEach(x => console.log(x))

// find：找到第一个满足条件的元素
arr.find(x => x > 2)           // 3

// some / every
arr.some(x => x > 3)           // true
arr.every(x => x > 0)          // true
```

## 8. 对象与遍历

对象用键值对存储数据，访问属性可用点号或方括号。

```
const user = {
  name: 'tom',
  age: 20
}

user.name           // tom
user['name']        // tom

// 遍历键
Object.keys(user)      // ["name", "age"]
Object.values(user)    // ["tom", 20]
Object.entries(user)   // [["name","tom"],["age",20]]

for (const key in user) {
  console.log(key, user[key])
}

// 解构
const { name, age } = user
```

## 9. 函数

函数是一等公民，可作为参数和返回值，支持声明式、表达式、箭头函数等多种定义方式。

```
// 函数声明
function add(a, b) {
  return a + b
}

// 函数表达式
const add2 = function (a, b) {
  return a + b
}

// 箭头函数
const add3 = (a, b) => a + b
const square = x => x * x   // 单参数可省略括号

// 默认参数
function greet(name = 'world') {
  return `Hello, ${name}`
}
```

## 10. 箭头函数与普通函数的区别

- 箭头函数没有自己的 this，this 继承外层作用域
- 箭头函数没有 arguments 对象
- 箭头函数不能用作构造函数

```
const obj = {
  name: 'tom',
  sayName: function () {
    console.log(this.name)   // this 指向 obj
  },
  sayArrow: () => {
    console.log(this.name)   // this 继承外层（window）
  }
}
```

## 11. 作用域与闭包

闭包：函数可以访问其外部函数的变量，即使外部函数已执行完毕。

```
function outer() {
  const count = 0
  return function () {
    return ++count   // 内部函数持有 count 引用
  }
}

const counter = outer()
counter()   // 1
counter()   // 2
counter()   // 3
```

## 12. this 指向

this 取决于调用方式。

```
const obj = {
  name: 'tom',
  show() {
    console.log(this.name)
  }
}

obj.show()        // tom（this 指向 obj）

const fn = obj.show
fn()              // undefined（this 指向全局，严格模式下）

// call / apply / bind 显式绑定
function greet() {
  console.log(this.name)
}
const user = { name: 'jerry' }
greet.call(user)      // jerry
greet.apply(user)     // jerry
const bound = greet.bind(user)
bound()               // jerry
```

## 13. 原型与原型链

每个对象都有 `__proto__` 指向其构造函数的 prototype，形成原型链。

```
function Person(name) {
  this.name = name
}

Person.prototype.sayHi = function () {
  console.log('Hi, ' + this.name)
}

const p = new Person('tom')
p.sayHi()                    // Hi, tom

p.__proto__ === Person.prototype      // true
Person.prototype.constructor === Person  // true
```

## 14. class 类

ES6 引入 class 语法糖，本质仍是原型。

```
class Animal {
  constructor(name) {
    this.name = name
  }

  speak() {
    console.log(this.name + ' makes a sound')
  }
}

class Dog extends Animal {
  constructor(name) {
    super(name)   // 调用父类构造
  }

  speak() {
    console.log(this.name + ' barks')
  }
}

const dog = new Dog('旺财')
dog.speak()   // 旺财 barks
```

## 15. Promise

处理异步操作，解决回调地狱。

```
const promise = new Promise((resolve, reject) => {
  setTimeout(() => {
    resolve('成功')
    // reject('失败')
  }, 1000)
})

promise
  .then(result => {
    console.log(result)      // 成功
    return 'next'
  })
  .then(val => console.log(val))   // next
  .catch(err => console.error(err))
  .finally(() => console.log('结束'))
```

## 16. async / await

Promise 的语法糖，让异步代码像同步一样书写。

```
function fetchData() {
  return new Promise(resolve => {
    setTimeout(() => resolve('data'), 1000)
  })
}

async function getData() {
  try {
    const data = await fetchData()
    console.log(data)
  } catch (err) {
    console.error(err)
  }
}

getData()
```

## 17. Promise 静态方法

这些静态方法用于并发控制，注意 all 遇错即拒，allSettled 等待全部完成。

```
// Promise.all：全部成功才成功
Promise.all([p1, p2, p3]).then(results => {})

// Promise.race：最快的先返回
Promise.race([p1, p2]).then(result => {})

// Promise.allSettled：全部完成，不管成功失败
Promise.allSettled([p1, p2]).then(results => {})

// Promise.any：任意一个成功
Promise.any([p1, p2]).then(result => {})
```

## 18. 事件循环 Event Loop

JavaScript 单线程，通过事件循环处理异步任务。微任务优先于宏任务执行。

- 宏任务：setTimeout、setInterval、I/O
- 微任务：Promise.then、queueMicrotask、MutationObserver

```
console.log('1')

setTimeout(() => console.log('2'), 0)   // 宏任务

Promise.resolve().then(() => console.log('3'))  // 微任务

console.log('4')

// 输出顺序：1 4 3 2
```

## 19. 解构赋值

解构能快速从数组或对象中提取变量，支持默认值和剩余参数。

```
// 数组解构
const [a, b, ...rest] = [1, 2, 3, 4]
// a=1, b=2, rest=[3,4]

// 对象解构
const { name, age = 18 } = { name: 'tom' }
// name='tom', age=18（默认值）

// 重命名
const { name: userName } = { name: 'tom' }
// userName='tom'
```

## 20. 展开运算符

展开运算符 ... 用于展开数组/对象，也可作为剩余参数收集多个值。

```
const arr1 = [1, 2]
const arr2 = [...arr1, 3, 4]     // [1, 2, 3, 4]

const obj1 = { a: 1 }
const obj2 = { ...obj1, b: 2 }   // { a: 1, b: 2 }

// 函数参数
function sum(...nums) {
  return nums.reduce((s, n) => s + n, 0)
}
sum(1, 2, 3, 4)   // 10
```

## 21. 模板字符串

用反引号包裹，支持变量插值和多行文本，比字符串拼接更清晰。

```
const name = 'tom'
const age = 20

const str = `我叫 ${name}，今年 ${age} 岁`
const multi = `
  多行
  字符串
`
```

## 22. Map 与 Set

Map 的键可为任意类型，Set 自动去重，是两种常用的集合数据结构。

```
// Map：键值对，键可为任意类型
const map = new Map()
map.set('a', 1)
map.set('b', 2)
map.get('a')        // 1
map.has('a')        // true
map.size            // 2
map.forEach((v, k) => console.log(k, v))

// Set：集合，元素唯一
const set = new Set([1, 2, 2, 3])
set.size            // 3（去重）
set.add(4)
set.has(1)          // true

// 数组去重
const unique = [...new Set([1, 2, 2, 3])]   // [1, 2, 3]
```

## 23. 深拷贝与浅拷贝

- 浅拷贝：只复制第一层，嵌套对象共享引用。
- 深拷贝：递归复制所有层级。

```
// 浅拷贝
const obj = { a: 1, b: { c: 2 } }
const shallow = { ...obj }
shallow.b.c = 3   // 影响原对象

// 深拷贝
const deep = JSON.parse(JSON.stringify(obj))
// 或 structuredClone
const deep2 = structuredClone(obj)
```

注意：JSON 方式无法拷贝函数、undefined、Date 等特殊值。

## 24. 防抖与节流

- 防抖：一段时间内多次触发只执行最后一次。
- 节流：一段时间内只执行一次。

```
// 防抖
function debounce(fn, delay) {
  let timer = null
  return function (...args) {
    clearTimeout(timer)
    timer = setTimeout(() => fn.apply(this, args), delay)
  }
}

// 节流
function throttle(fn, delay) {
  let last = 0
  return function (...args) {
    const now = Date.now()
    if (now - last >= delay) {
      last = now
      fn.apply(this, args)
    }
  }
}
```

## 25. 模块化 ESM

ES6 模块通过 export 导出、import 导入，支持默认导出和命名导出。

```
// math.js
export const add = (a, b) => a + b
export default function multiply(a, b) {
  return a * b
}

// main.js
import multiply, { add } from './math.js'

console.log(add(1, 2))        // 3
console.log(multiply(2, 3))   // 6

// 动态导入
import('./math.js').then(mod => {
  console.log(mod.add(1, 1))
})
```

## 26. 可选链与空值合并

```
const user = {
  info: {
    name: 'tom'
  }
}

// 可选链：属性不存在返回 undefined，不报错
user.info?.name       // "tom"
user.info?.age        // undefined
user.address?.city    // undefined

// 空值合并：左侧为 null/undefined 时取右侧
const name = user.name ?? '默认名'    // '默认名'
const count = 0 ?? 10                 // 0（0 不是 null/undefined）
const count2 = 0 || 10                // 10（|| 会判断假值）
```

## 27. Symbol 与迭代器

Symbol 生成唯一标识，实现迭代器协议后可被 for...of 遍历。

```
// Symbol 唯一值
const s1 = Symbol('x')
const s2 = Symbol('x')
s1 === s2   // false

// 迭代器协议
const obj = {
  [Symbol.iterator]() {
    let i = 0
    return {
      next() {
        i++
        return i <= 3
          ? { value: i, done: false }
          : { done: true }
      }
    }
  }
}

for (const v of obj) {
  console.log(v)   // 1 2 3
}
```

## 28. 错误处理

try/catch 捕获异常，finally 总是执行，也可自定义错误类型。

```
try {
  throw new Error('出错了')
} catch (err) {
  console.error(err.message)
} finally {
  console.log('总是执行')
}

// 自定义错误
class MyError extends Error {
  constructor(message) {
    super(message)
    this.name = 'MyError'
  }
}
```

## 29. 常用数组排序与查找

sort 默认按字符串排序，传入比较函数才能正确比较数字。

```
const arr = [5, 3, 8, 1]

// 升序排序
arr.sort((a, b) => a - b)      // [1, 3, 5, 8]

// 降序
arr.sort((a, b) => b - a)      // [8, 5, 3, 1]

// 对象数组排序
const users = [{ age: 30 }, { age: 20 }]
users.sort((a, b) => a.age - b.age)
```

## 30. JSON 操作

JSON 是通用的数据交换格式，stringify 序列化、parse 反序列化。

```
const obj = { name: 'tom', age: 20 }

// 序列化
const json = JSON.stringify(obj)      // '{"name":"tom","age":20}'

// 反序列化
const parsed = JSON.parse(json)       // { name: 'tom', age: 20 }

// 自定义序列化
JSON.stringify(obj, ['name'])         // 只序列化 name
JSON.stringify(obj, null, 2)          // 缩进 2 空格
```

## 31. 面向对象与继承方式

JS 继承有原型链和 class 两种方式，class 是原型链的语法糖，写法更简洁。

```
// 1. 原型链继承
function Parent() { this.name = 'parent' }
function Child() {}
Child.prototype = new Parent()

// 2. class 继承（推荐）
class ParentClass {
  constructor(name) { this.name = name }
  greet() { console.log('Hi ' + this.name) }
}

class ChildClass extends ParentClass {
  constructor(name, age) {
    super(name)
    this.age = age
  }
}
```

## 32. 内存管理与垃圾回收

JavaScript 通过引用计数和标记清除自动回收内存。

```
// 引用计数：无引用的对象被回收
let obj = { data: 1 }
obj = null   // 原对象无引用，可被回收

// 标记清除：从根对象出发，无法到达的对象被回收
// 闭包中若持有大对象引用，需注意及时释放
```

## 33. 常用内置对象

Math、Date、Number 是常用的内置对象，掌握常用方法即可满足大部分场景。

```
// Math：数学计算
Math.max(1, 2, 3)     // 3，取最大值
Math.min(1, 2, 3)     // 1，取最小值
Math.floor(3.9)       // 3，向下取整
Math.ceil(3.1)        // 4，向上取整
Math.round(3.5)       // 4，四舍五入（.5 会进位）
Math.abs(-5)          // 5，取绝对值
Math.random()         // 返回 0~1 之间的随机小数

// Date：日期时间
const now = new Date()   // 当前时间
now.getFullYear()        // 2026，年份
now.getMonth()           // 0~11，月份（要 +1 才是真实月份）
now.getDate()            // 几号（1~31）
now.getDay()             // 星期几（0 表示周日）
now.getHours()           // 小时
now.getTime()            // 毫秒时间戳（从 1970-01-01 起算）

// Number：数字处理
Number.isInteger(10)  // true，判断是否为整数
Number.isNaN(NaN)     // true，判断是否为 NaN
parseInt('10px')      // 10，解析整数，遇到非数字停止
parseFloat('3.14')    // 3.14，解析浮点数
```
