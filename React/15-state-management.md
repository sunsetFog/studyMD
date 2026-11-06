# 15 - 状态管理总览与分层

## 1. 状态分类学（先分类再选工具）

| 类别 | 示例 | 正确归属 |
|---|---|---|
| 组件本地 | 弹窗开关、输入草稿 | useState / useReducer |
| 跨组件共享（低频） | 主题、当前用户、i18n | Context |
| 跨组件共享（高频） | 购物车实时数量、画布状态 | Zustand / Jotai |
| 服务端数据 | 列表、详情、字典 | **TanStack Query（服务端状态≠客户端状态）** |
| URL 状态 | 筛选、分页、tab | 路由 query |
| 全局副作用派生 | 未读数、购物车总额 | 从源状态计算/订阅 |

最大误区：**把服务端数据塞进 Redux/Zustand**——缓存、失效、重取、竞态这些服务端数据特有问题，客户端状态库并不擅长。React 生态对此的标准答案是 Query 层。

## 2. React 状态方案地图

| 方案 | 定位 | 适用 |
|---|---|---|
| useState/useReducer | 组件本地 | 默认起点 |
| Context | 低频全局值 | 主题/登录态（见 07） |
| **Zustand** | 轻量全局 store | 中小型全局状态首选 |
| **Jotai** | 原子化状态 | 细粒度依赖图 |
| **Redux Toolkit** | 集中式 + 中间件生态 | 大型团队、严格规范、devtools 重度 |
| TanStack Query | 服务端状态缓存 | 接口数据（几乎是必选） |
| React Hook Form | 表单状态 | 表单域 |
| URL（Router） | 可分享视图状态 | 筛选/分页 |

## 3. 决策树

```
1. 只有一个组件用？ → useState
2. 父子/兄弟几层内？ → props 提升
3. 低频全局值？ → Context
4. 服务端数据？ → TanStack Query
5. 高频共享/复杂全局？ → Zustand（默认）/ Jotai（原子粒度）
6. 团队需要强规范集中式？ → Redux Toolkit
7. 能放 URL 的视图状态？ → 一律 URL
```

经验法则：**大多数现代 React 项目 = 组件 state + Query + Zustand + URL**，Redux 用于存量或强规范团队。

## 4. 服务端状态 vs 客户端状态（关键认知）

服务端数据的特殊性：

```
1. 有缓存与新鲜度概念（staleTime）
2. 有失效与重取（invalidate）
3. 有并发请求去重（同一 key 只发一次）
4. 有竞态（旧响应覆盖新响应）
5. 有分页/无限加载/乐观更新等模式
```

手写 useState + useEffect 管理它们 = 重复实现半个数据库，且每个项目实现质量参差。**默认答案：useQuery/useMutation**（见 19）。

## 5. 状态设计原则

| 原则 | 说明 |
|---|---|
| 单一数据源 | 一个状态只存一处，派生值计算 |
| 最小状态集 | 能算的不存（页面列表由 page+filter 推导） |
| 不可变更新 | 全生态默认假设（StrictMode/imMemo/Query 依赖 Object.is） |
| 状态机思维 | 互斥态用联合类型，非法态不可表示（见 04/08） |
| 归属就近 | 能放组件不放全局；全局是最后选择 |

## 6. 典型组合示例（电商购物车页）

```tsx
// URL：分页/筛选（可分享）
const [searchParams, setSearchParams] = useSearchParams()

// 服务端：商品列表
const { data } = useQuery({ queryKey: ['products', searchParams.toString()], … })

// 全局客户端：购物车
const items = useCartStore(s => s.items)

// 本地：选中行
const [selected, setSelected] = useState<Set<number>>(new Set())

// 派生：合计（不存 state）
const total = useMemo(() => items.reduce((s, i) => s + i.price * i.qty, 0), [items])
```

## 7. 持久化

| 层 | 工具 | 注意 |
|---|---|---|
| URL | searchParams | 筛选/分页首选 |
| localStorage | zustand persist / useLocalStorage hook | 白名单 + 版本迁移 + 脱敏 |
| sessionStorage | 同上 | 草稿类 |

localStorage 写入注意 JSON 版本字段（schema 变更迁移），敏感信息不落盘（见 34）。

## 8. 状态管理反模式速判

```
反模式：所有 state 进 Redux（Redux 当数据库）
反模式：Context 高频更新（输入/动画）
反模式：服务端数据手写 effect 管理（无缓存无失效）
反模式：父子互传 5 层 props 不肯提升/建 store
反模式：镜像 state（props 拷贝进 useState）
反模式：能算的存两份（total 存 state）
```

## 常见坑

1. **一个 store 巨无霸**：按业务域拆 store。
2. **服务端数据失效缺失**：写操作后忘记 invalidate 相关 query。
3. **URL 状态复杂对象直接存**：序列化加版本与解析兜底。
4. **派生值存 state**：两处数据源漂移。
5. **跨 store 循环依赖**：action 内互相调用而非顶层 import。

## Checklist

- [ ] 状态分类清晰，服务端数据走 Query
- [ ] URL 承载可分享视图状态
- [ ] 无镜像/冗余状态
- [ ] 高频状态不在 Context
- [ ] 持久化白名单 + 脱敏
- [ ] 方案选型有团队共识文档
