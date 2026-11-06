# 27 · 邮件发送、文件上传与对象存储

## 一、邮件发送

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

```yaml
spring:
  mail:
    host: smtp.exmail.qq.com
    port: 465
    username: noreply@example.com
    password: ${SMTP_PASSWORD}      # 授权码/SMTP 密码，走环境变量
    properties:
      mail.smtp.auth: true
      mail.smtp.ssl.enable: true    # 465 端口
```

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MailService {

    private final JavaMailSender mailSender;

    @Value("${spring.mail.username}")
    private String from;

    /** 简单文本 */
    public void sendText(String to, String subject, String content) {
        SimpleMailMessage msg = new SimpleMailMessage();
        msg.setFrom(from);
        msg.setTo(to);
        msg.setSubject(subject);
        msg.setText(content);
        mailSender.send(msg);
    }

    /** HTML + 附件 */
    public void sendHtml(String to, String subject, String htmlBody, File attachment) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
            helper.setFrom(from);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlBody, true);          // true = HTML
            if (attachment != null) {
                helper.addAttachment(attachment.getName(), attachment);
            }
            mailSender.send(message);
        } catch (MessagingException e) {
            throw new BusinessException(500, "邮件发送失败");
        }
    }
}
```

**实践要点**：
- 发送耗时（1-3s）→ 走 @Async 或 MQ，别阻塞业务请求
- 失败重试 + 落库记录（审计 + 重发）
- HTML 内容模板化（Thymeleaf 模板拼接）

## 二、文件上传

```java
@RestController
@RequestMapping("/api/files")
@RequiredArgsConstructor
public class FileController {

    private final FileStorageService storageService;

    @PostMapping("/upload")
    public Result<String> upload(@RequestParam("file") MultipartFile file,
                                 @RequestParam String scene) {
        return Result.ok(storageService.store(file, scene));
    }
}
```

```java
@Service
public class FileStorageService {

    private static final List<String> ALLOWED_EXT = List.of("jpg", "png", "pdf", "xlsx");
    private static final long MAX_SIZE = 10 * 1024 * 1024L;

    public String store(MultipartFile file, String scene) {
        if (file.isEmpty()) throw new BusinessException(400, "文件为空");
        if (file.getSize() > MAX_SIZE) throw new BusinessException(400, "文件超过 10MB");

        String ext = FilenameUtils.getExtension(file.getOriginalFilename());
        if (!ALLOWED_EXT.contains(ext.toLowerCase())) throw new BusinessException(400, "不支持的类型");

        // 关键：重命名，防止路径穿越与覆盖（绝不使用用户原始文件名拼路径）
        String key = scene + "/" + LocalDate.now() + "/" + UUID.randomUUID() + "." + ext;
        Path target = Paths.get("/data/upload").resolve(key).normalize();
        if (!target.startsWith("/data/upload")) throw new BusinessException(400, "非法路径");

        try {
            Files.createDirectories(target.getParent());
            file.transferTo(target);
        } catch (IOException e) {
            throw new BusinessException(500, "保存失败");
        }
        return "/files/" + key;
    }
}
```

```yaml
spring:
  servlet:
    multipart:
      max-file-size: 10MB
      max-request-size: 20MB
```

## 三、对象存储（MinIO / OSS / COS）

```xml
<dependency>
    <groupId>io.minio</groupId>
    <artifactId>minio</artifactId>
    <version>8.5.10</version>
</dependency>
```

```yaml
oss:
  endpoint: https://oss.example.com
  access-key: ${OSS_AK}
  secret-key: ${OSS_SK}
  bucket: app-files
```

```java
@Service
@RequiredArgsConstructor
public class OssService {

    private final MinioClient minioClient;

    @Value("${oss.bucket}")
    private String bucket;

    public String upload(MultipartFile file, String scene) throws Exception {
        String key = scene + "/" + UUID.randomUUID() + "-"
                + FilenameUtils.getName(file.getOriginalFilename());
        minioClient.putObject(PutObjectArgs.builder()
                .bucket(bucket)
                .object(key)
                .stream(file.getInputStream(), file.getSize(), -1)
                .contentType(file.getContentType())
                .build());
        return key;
    }

    /** 预签名 URL：私有 bucket 限时访问，前端直传/直下载 */
    public String presignedUrl(String key, int expireMinutes) throws Exception {
        return minioClient.getPresignedObjectUrl(GetPresignedObjectUrlArgs.builder()
                .method(Method.GET)
                .bucket(bucket)
                .object(key)
                .expiry(expireMinutes * 60)
                .build());
    }
}
```

**云上最佳实践：前端直传**（服务端不中转大文件）：

```
1. 前端请求后端：POST /api/files/presign  { scene, filename, size }
2. 后端校验 + 生成 key，返回预签名 PUT URL
3. 前端直接 PUT 到 OSS
4. 前端回传 key，后端落库文件记录
```

好处：绕过应用服务器带宽与内存；云端（阿里 OSS/Tencent COS/CVE 的 S3）同思路，用各自 SDK 的 presign。

## 四、文件安全清单

- 文件名/路径服务端生成，白名单扩展名
- Content-Type 校验（魔数校验更稳）
- 上传目录不可执行（nginx location 禁 php/exe 直出）
- 私有桶 + 预签名 URL 限时访问
- 图片服务端压缩/格式转换（省带宽 + 防恶意文件）

## 常见坑

- **坑 1**：MultipartFile.transferTo 用相对路径 → 相对于临时目录，找不到文件
- **坑 2**：临时目录（/tmp）空间不足 → 部署时设 `server.tomcat.basedir` 或指定 multipart location
- **坑 3**：上传大文件用 MultipartFile.readAllBytes → OOM；流式处理
- **坑 4**：邮件发件人伪造域拦截 → 配置 SPF/DKIM，用企业 SMTP
- **坑 5**：邮件同步发送拖慢接口 → 异步化

## Checklist

- [ ] 上传白名单 + 重命名 + 大小限制
- [ ] 生产文件走对象存储私有桶 + 预签名
- [ ] 大文件前端直传方案
- [ ] 邮件异步化 + 发送记录落库
