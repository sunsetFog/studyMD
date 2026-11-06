# 22 · 接口文档：SpringDoc OpenAPI

> Springfox 已停更且不兼容 Boot 3，**Boot 3 统一使用 springdoc-openapi**。

## 一、集成

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

自动获得：
- Swagger UI：`http://localhost:8080/swagger-ui/index.html`
- OpenAPI JSON：`/v3/api-docs`
- YAML：`/v3/api-docs.yaml`

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
    enabled: true           # 生产可关
  swagger-ui:
    path: /swagger-ui.html
  packages-to-scan: com.example.demo.controller
```

## 二、全局信息

```java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI openAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("商城系统 API")
                        .description("SpringBoot 3 后端接口文档")
                        .version("v1.0"))
                .components(new Components().addSecuritySchemes("bearer-jwt",
                        new SecurityScheme()
                                .type(SecurityScheme.Type.HTTP)
                                .scheme("bearer")
                                .bearerFormat("JWT")))
                .addSecurityItem(new SecurityRequirement().addList("bearer-jwt"));
    }
}
```

配置全局鉴权后，Swagger UI 右上角 Authorize 填 token，所有接口请求自动带 Authorization 头。

## 三、Controller 注解

```java
@Tag(name = "用户管理", description = "用户 CRUD 接口")
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @Operation(summary = "分页查询用户", description = "支持按用户名模糊、状态过滤")
    @GetMapping
    public Result<PageResult<UserDTO>> list(
            @Parameter(description = "页码，从 1 开始", example = "1") @RequestParam int pageNum,
            @RequestParam int pageSize) { ... }

    @Operation(summary = "创建用户")
    @ApiResponses({
        @ApiResponse(responseCode = "200", description = "成功"),
        @ApiResponse(responseCode = "400", description = "参数错误")
    })
    @PostMapping
    public Result<Long> create(@RequestBody UserCreateDTO dto) { ... }
}
```

## 四、DTO 注解

```java
@Schema(description = "创建用户请求")
@Data
public class UserCreateDTO {

    @Schema(description = "用户名", example = "tom", requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank
    private String username;

    @Schema(description = "手机号", example = "13800138000")
    private String phone;

    @Schema(description = "状态：0-禁用 1-启用", allowableValues = {"0", "1"})
    private Integer status;
}
```

## 五、分组

```java
@Bean
public GroupedOpenApi adminApi() {
    return GroupedOpenApi.builder()
            .group("admin")
            .pathsToMatch("/api/admin/**")
            .build();
}

@Bean
public GroupedOpenApi appApi() {
    return GroupedOpenApi.builder()
            .group("app")
            .pathsToMatch("/api/v1/**")
            .build();
}
```

## 六、文档即契约的协作流程

1. 后端定义 DTO + 注解，代码先行
2. 导出 `/v3/api-docs` 供前端生成 TS 类型（openapi-typescript）
3. 接口变更通过文档 diff 评审
4. 生产环境 `springdoc.enabled=false` 关闭（或加权限）

## 常见坑

- **坑 1**：引入 springfox 依赖在 Boot 3 报 `Failed to start bean documentationPluginsBootstrapper` → 移除，换 springdoc
- **坑 2**：统一响应 Result<T> 泛型丢失 → 返回类型写 `Result<PageResult<UserDTO>>` 而不是 `Result<Object>`
- **坑 3**：Security 把 swagger 路径拦截 → 白名单加 `/v3/api-docs/**`、`/swagger-ui/**`
- **坑 4**：文档暴露生产 → 生产关闭或网关屏蔽
- **坑 5**：枚举/状态码含义不写 → 文档可读性差，字段必填与示例要补齐

## Checklist

- [ ] springdoc 集成 + 全局 JWT 安全配置
- [ ] 每个 Controller 有 @Tag/@Operation
- [ ] DTO 字段有 @Schema 描述与示例
- [ ] 生产环境关闭文档
