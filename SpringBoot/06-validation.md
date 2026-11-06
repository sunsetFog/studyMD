# 06 · 参数校验

## 一、依赖与快速上手

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

> SpringBoot 3 用 Jakarta 命名空间：`jakarta.validation.*`。

```java
@Data
public class UserCreateDTO {

    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度 2-20")
    private String username;

    @NotNull(message = "手机号不能为空")
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式错误")
    private String phone;

    @Email(message = "邮箱格式错误")
    private String email;

    @Min(value = 0, message = "年龄不能为负")
    @Max(150)
    private Integer age;
}
```

```java
@PostMapping("/users")
public Result<Long> create(@RequestBody @Valid UserCreateDTO dto) { ... }
```

## 二、常用注解速查

| 注解 | 说明 |
|---|---|
| @NotNull / @NotBlank / @NotEmpty | 非 null / 非 null+非空白（String）/ 非 null+非空（集合） |
| @Size | 长度（String/集合/数组/Map） |
| @Min / @Max / @Range | 数值范围 |
| @Email / @Pattern | 邮箱 / 正则 |
| @Positive / @PositiveOrZero | 正数 |
| @Past / @Future | 过去/未来时间 |
| @AssertTrue | 布尔为 true（用于关联字段校验） |

**三兄弟区别**：String 用 `@NotBlank`；集合用 `@NotEmpty`；非字符串非集合（如 Integer）用 `@NotNull`。

## 三、嵌套对象与集合校验

```java
public class OrderDTO {
    @Valid @NotNull
    private AddressDTO address;          // 嵌套对象：必须加 @Valid 才级联校验

    @NotEmpty
    @Valid                               // List 元素级联
    private List<OrderItemDTO> items;

    @Size(min = 1)
    private List<@NotBlank String> tags; // 元素级简单校验
}
```

## 四、分组校验（同一个 DTO，不同场景不同规则）

```java
public class UserDTO {
    public interface Create {}
    public interface Update {}

    @Null(groups = Create.class)          // 创建时禁止传 id
    @NotNull(groups = Update.class)       // 更新时必须传 id
    private Long id;

    @NotBlank(groups = {Create.class, Update.class})
    private String username;
}

@PostMapping
public Result<Long> create(@RequestBody @Validated(UserDTO.Create.class) UserDTO dto) {...}

@PutMapping
public Result<Void> update(@RequestBody @Validated(UserDTO.Update.class) UserDTO dto) {...}
```

> 注意：`@Valid` 不支持分组，分组必须用 `@Validated`（Spring 扩展）。

## 五、自定义校验器

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface Phone {
    String message() default "手机号格式错误";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class PhoneValidator implements ConstraintValidator<Phone, String> {
    private static final Pattern P = Pattern.compile("^1[3-9]\\d{9}$");
    @Override
    public boolean isValid(String value, ConstraintValidatorContext ctx) {
        if (value == null) return true;   // 空值交给 @NotNull 管
        return P.matcher(value).matches();
    }
}
```

## 六、校验失败的处理（衔接 07 篇）

MethodArgumentNotValidException 抛出后由全局异常处理器兜住：

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public Result<Void> handleValid(MethodArgumentNotValidException e) {
    String msg = e.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .collect(Collectors.joining("; "));
    return Result.fail(400, msg);
}
```

## 七、手动校验与 Service 层校验

```java
@Autowired
private Validator validator;

Set<ConstraintViolation<UserDTO>> violations = validator.validate(dto);
if (!violations.isEmpty()) throw new BadRequestException(...);
```

**校验分层原则**：
- Controller：格式/非空/长度（@Valid）
- Service：业务规则（用户名是否已存在、库存是否足够）——**业务校验不能只靠注解**

## 常见坑

- **坑 1**：加了 @Valid 但 DTO 内嵌套对象没加 → 内层不校验
- **坑 2**：方法参数级校验失效 → Controller 类上没加 `@Validated`（校验 @RequestParam/@PathVariable 需类级 @Validated，且抛 ConstraintViolationException）
- **坑 3**：@NotBlank 用在 Integer 上 → 运行时异常，Integer 用 @NotNull
- **坑 4**：依赖缺失静默不校验 → 确认引入 validation starter

## Checklist

- [ ] 所有入参 DTO 都有校验注解，Controller 加 @Valid
- [ ] Create/Update 共用 DTO 时用分组
- [ ] 自定义校验器已沉淀（手机号、身份证等）
- [ ] 业务唯一性校验在 Service 层（配合唯一索引兜底）
