# 05 · RESTful API 设计

## 一、核心思想

**资源（Resource）为中心**：URL 表示资源，HTTP 方法表示操作，状态码表示结果。

```
GET    /api/users        # 用户列表
GET    /api/users/1      # 单个用户
POST   /api/users        # 创建用户
PUT    /api/users/1      # 全量更新
PATCH  /api/users/1      # 部分更新
DELETE /api/users/1      # 删除用户
```

## 二、URL 设计规范

| 规则 | 好 | 坏 |
|---|---|---|
| 名词复数 | `/users` | `/getUserList` |
| 层级表从属 | `/users/1/orders` | `/getOrdersByUserId?uid=1` |
| 动作用方法 | `PUT /users/1` | `/users/update/1` |
| 小写连字符 | `/order-items` | `/orderItems` |
| 过滤参数 | `/users?status=1&page=2` | `/users/status/1/page/2` |

避免层级过深：超过两层用查询参数，如 `/comments?postId=1`。

## 三、状态码语义

| 码 | 场景 |
|---|---|
| 200 | 成功（含业务失败时的统一 200 + code 字段方案） |
| 201 | 创建成功（POST） |
| 204 | 删除成功无内容 |
| 400 | 参数错误 |
| 401 | 未认证（没登录/token 无效） |
| 403 | 已认证但无权限 |
| 404 | 资源不存在 |
| 409 | 冲突（重复创建） |
| 422/400 | 校验失败 |
| 500 | 服务器错误 |

**两种流派**：
- 严格 REST：HTTP 状态码承载全部语义（适合开放 API）
- 国内主流：**HTTP 恒 200（除 401），业务语义放 body.code**（配合统一响应体，见 07 篇）——前端拦截器处理简单。本文档系列采用后者。

## 四、统一响应体设计

```java
@Data
public class Result<T> {
    private int code;        // 0 成功，非 0 业务错误码
    private String message;
    private T data;
    private long timestamp;

    public static <T> Result<T> ok(T data) {
        Result<T> r = new Result<>();
        r.code = 0; r.message = "success"; r.data = data;
        r.timestamp = System.currentTimeMillis();
        return r;
    }

    public static <T> Result<T> fail(int code, String message) { ... }
}
```

**分页约定**：

```json
{
  "code": 0,
  "data": {
    "list": [...],
    "total": 123,
    "pageNum": 1,
    "pageSize": 10
  }
}
```

## 五、版本管理

- URL 版本（主流）：`/api/v1/users`
- Header 版本：`Accept: application/vnd.demo.v1+json`

## 六、Controller 示例（完整风格）

```java
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @GetMapping
    public Result<PageResult<UserDTO>> list(UserQueryDTO query) {
        return Result.ok(userService.page(query));
    }

    @GetMapping("/{id}")
    public Result<UserDTO> detail(@PathVariable Long id) {
        return Result.ok(userService.getById(id));
    }

    @PostMapping
    public Result<Long> create(@RequestBody @Valid UserCreateDTO dto) {
        return Result.ok(userService.create(dto));
    }

    @PutMapping("/{id}")
    public Result<Void> update(@PathVariable Long id,
                               @RequestBody @Valid UserUpdateDTO dto) {
        dto.setId(id);
        userService.update(dto);
        return Result.ok(null);
    }

    @DeleteMapping("/{id}")
    public Result<Void> delete(@PathVariable Long id) {
        userService.delete(id);
        return Result.ok(null);
    }
}
```

## 七、幂等性

| 方法 | 幂等 | 说明 |
|---|---|---|
| GET/DELETE/PUT | 是 | 重复调用结果相同 |
| POST | 否 | 需业务侧防重：唯一索引 / 幂等 token / 分布式锁 |

## 常见坑

- **坑 1**：DELETE/PUT 有些网关/老客户端不支持 → 提供 `POST /users/1/delete` 兜底
- **坑 2**：REST 派与统一 200 派混用 → 团队定一套写进接口文档规范
- **坑 3**：URL 里带动词 `getUser` → 改资源名词 + 方法语义
- **坑 4**：分页参数命名混乱 → 统一 `pageNum/pageSize`（或 `page/size`）全项目一致

## Checklist

- [ ] URL 全部资源化名词，操作靠 HTTP 方法
- [ ] 统一响应体 + 分页结构已沉淀到 common 包
- [ ] 错误码表有文档（业务码分段：如 1xxx 用户、2xxx 订单）
- [ ] POST 接口有幂等防护策略
