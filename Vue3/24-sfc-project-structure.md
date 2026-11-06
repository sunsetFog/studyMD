# 24 - SFC 与项目结构

## 1. SFC 规范

```vue
<!-- 标准模板：块顺序 template → script → style -->
<script setup lang="ts">
// 1. 类型 import
import type { PropType } from 'vue'
// 2. 普通 import（vue → 第三方 → @/ 别名 → 相对）
import { ref, computed } from 'vue'
import dayjs from 'dayjs'
import { useUserStore } from '@/stores/user'
import ChildComp from './ChildComp.vue'
// 3. define 宏（defineProps/defineEmits/defineOptions…）
// 4. composables 调用
// 5. 响应式状态
// 6. computed
// 7. 函数（methods）
// 8. 生命周期
// 9. defineExpose
</script>
```

统一顺序提高可读性，ESLint（vue/block-order 等规则）固化。

## 2. 推荐目录结构

```
src/
├── main.ts
├── App.vue
├── router/
│   ├── index.ts
│   ├── routes/            # 路由表按模块拆
│   └── guards.ts
├── stores/                # Pinia
├── api/                   # 接口层（按业务域）
│   ├── user.ts
│   └── order.ts
├── composables/           # 通用 + 业务 composables
├── components/
│   ├── common/            # 通用展示组件（Button/Modal）
│   ├── business/          # 业务组件（UserCard/OrderItem）
│   └── layout/            # 布局组件
├── views/                 # 页面（对应路由）
│   ├── user/
│   │   ├── UserList.vue
│   │   └── components/    # 页面私有子组件
│   └── order/
├── directives/
├── utils/                 # 纯函数工具（无状态）
├── hooks/                 # （与 composables 合并则删）
├── types/                 # 全局类型声明
├── constants/             # 常量/枚举
└── assets/
```

规则：
- **views 页面私有组件就近放** `views/xxx/components/`，晋升为通用组件才移入 components/business
- **api 层与类型同域**：`api/order.ts` 导出接口函数 + 参数/返回类型
- **utils 无状态无副作用**，含状态逻辑进 composables

## 3. 命名规范

| 类型 | 规范 | 示例 |
|---|---|---|
| 组件文件 | PascalCase 多词 | `UserCard.vue`（避免单词 `Card.vue` 防与 HTML 冲突） |
| 页面 | PascalCase + View 后缀可选 | `UserList.vue` / `UserListView.vue` |
| composables | useXxx | `useAuth.ts` |
| 常量 | UPPER_SNAKE | `MAX_PAGE_SIZE` |
| 类型/接口 | PascalCase | `OrderItem`（接口不加 I 前缀） |
| CSS 类 | kebab-case / BEM | `.user-card__title` |

## 4. 路径别名

```ts
// vite.config.ts
resolve: { alias: { '@': fileURLToPath(new URL('./src', import.meta.url)) } }
// tsconfig.json
"paths": { "@/*": ["./src/*"] }
```

层级约定：`@/` 只允许向下引用，禁止 `views/A.vue` 引 `views/B/` 内部组件（跨页面复用走 components/business）。

## 5. 组件文件粒度

| 场景 | 粒度 |
|---|---|
| 页面 | 一个页面一个 .vue，>500 行拆子组件 |
| 子组件拆分信号 | 逻辑块独立（筛选区/表格区/分页区）、需复用、需独立测试 |
| 过度拆分 | 20 行的小组件满天飞，跳转成本 > 复用收益 |

## 6. 常量与枚举

```ts
// constants/order.ts
export const ORDER_STATUS = {
  pending: 0,
  paid: 1,
  shipped: 2,
} as const
export type OrderStatus = typeof ORDER_STATUS[keyof typeof ORDER_STATUS]

// 集中导出中文映射（避免模板里写魔法数字三元）
export const ORDER_STATUS_TEXT: Record<OrderStatus, string> = {
  0: '待付款', 1: '已付款', 2: '已发货',
}
```

## 7. 代码规范工具链

```bash
npm i -D eslint @vue/eslint-config-typescript prettier eslint-plugin-vue
```

要点：
- ESLint（vue/essential + ts 规则）管质量，Prettier 管格式，二者分工别混
- `eslint-plugin-vue` 选 flat config（3.4+ 时代标配）
- lint-staged + husky：commit 前只 lint 暂存文件
- VSCode 保存自动修复统一（.vscode/settings.json 入库）

## 8. 环境变量

```bash
# .env.development / .env.production
VITE_API_BASE=/api
VITE_APP_ENV=dev
```

```ts
const base = import.meta.env.VITE_API_BASE    // 客户端可读，VITE_ 前缀才暴露
// 类型增强 env.d.ts
interface ImportMetaEnv { readonly VITE_API_BASE: string }
```

## 常见坑

1. **单词组件名与原生元素冲突**（`Menu.vue`、`Header.vue`）：必须多词。
2. **views 跨页面互相 import**：产生循环依赖与隐式耦合。
3. **utils 里塞请求/状态**：utils 应纯净。
4. **别名没配 tsconfig paths**：TS 报错 can't find module。
5. **组件文件超 800 行**：难维护，按逻辑块拆。
6. **env 文件入库带密钥**：真实密钥走服务端注入，前端只有公开配置。

## Checklist

- [ ] 目录按职责分层，页面私有组件就近
- [ ] 组件命名多词 PascalCase
- [ ] @ 别名 + tsconfig paths 同步配置
- [ ] ESLint + Prettier + lint-staged 就位
- [ ] 常量/枚举集中管理，无魔法数字
- [ ] script setup 内代码块顺序统一
