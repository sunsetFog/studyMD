# 65 · CI/CD 流水线

## 1. 概念

| 术语 | 含义 |
|---|---|
| **CI**（持续集成） | 频繁合并代码 → 自动构建、测试、静态检查 |
| **CD**（持续交付/部署） | 自动化发布到测试/生产环境 |
| Pipeline | 流水线（阶段化的任务序列） |
| Artifact | 构建产物（jar/镜像） |
| Gate | 质量门禁（测试覆盖率、扫描结果） |

## 2. 流水线典型阶段

```
代码提交 → 触发 → 检出代码 → 编译 → 单元测试 → 静态扫描 → 构建镜像
  → 推送仓库 → 部署到 dev → 自动化测试 → 部署到 test → 人工审批
  → 部署到 prod（灰度/蓝绿） → 健康检查 → 通知
```

## 3. GitHub Actions

```yaml
# .github/workflows/build.yml
name: CI/CD
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven          # ★ 依赖缓存

      - name: Build & Test
        run: mvn -B clean verify          # verify 会跑 test + 集成测试

      - name: SonarQube Scan
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn -B sonar:sonar -Dsonar.projectKey=order

      - name: Build image
        run: |
          docker build -t ${{ secrets.REGISTRY }}/order:${{ github.sha }} .

      - name: Login & Push
        run: |
          echo ${{ secrets.REGISTRY_PWD }} | docker login ${{ secrets.REGISTRY }} -u ${{ secrets.REGISTRY_USER }} --password-stdin
          docker push ${{ secrets.REGISTRY }}/order:${{ github.sha }}

      - name: Deploy
        if: github.ref == 'refs/heads/main'
        run: |
          kubectl set image deployment/order order=${{ secrets.REGISTRY }}/order:${{ github.sha }} -n prod
          kubectl rollout status deployment/order -n prod --timeout=180s
```

> 密钥统一放 `secrets`，**绝不**写入代码库。

## 4. GitLab CI

```yaml
# .gitlab-ci.yml
stages: [build, test, package, deploy]

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"

cache:
  paths: [.m2/repository/]

build:
  stage: build
  image: maven:3.9-eclipse-temurin-17
  script: mvn -B compile

test:
  stage: test
  image: maven:3.9-eclipse-temurin-17
  script: mvn -B test
  artifacts:
    reports:
      junit: target/surefire-reports/*.xml

package:
  stage: package
  image: docker:24
  services: [docker:24-dind]
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy:
  stage: deploy
  script: ./deploy.sh $CI_COMMIT_SHA
  environment: {name: production}
  when: manual            # 人工确认
  only: [main]
```

## 5. Jenkins（Jenkinsfile）

```groovy
pipeline {
    agent any
    tools { maven 'maven3.9'; jdk 'jdk17' }
    environment {
        IMAGE = "registry.example.com/order:${env.GIT_COMMIT}"
    }
    stages {
        stage('Checkout') { steps { checkout scm } }
        stage('Build')    { steps { sh 'mvn -B clean package -DskipTests' } }
        stage('Test')     { steps { sh 'mvn -B test'
                                    junit 'target/surefire-reports/*.xml' } }
        stage('Sonar')    { steps { sh 'mvn sonar:sonar' } }
        stage('Image')    { steps { sh 'docker build -t $IMAGE . && docker push $IMAGE' } }
        stage('Deploy')   {
            when { branch 'main' }
            steps {
                input message: '确认发布生产？'
                sh 'kubectl set image deployment/order order=$IMAGE -n prod'
            }
        }
    }
    post {
        success { dingtalk(robot: 'ci', type: 'TEXT', text: ['构建成功']) }
        failure { dingtalk(robot: 'ci', type: 'TEXT', text: ['构建失败']) }
    }
}
```

## 6. Maven 多模块与跳过测试

```bash
mvn -B clean verify                    # 跑全部测试
mvn -B clean package -DskipTests       # 跳过测试
mvn -B clean package -Dmaven.test.skip=true   # 连编译都不编译测试
mvn -pl order-service -am clean install       # 只构建指定模块及其依赖
```

## 7. 质量门禁

| 检查 | 工具 |
|---|---|
| 单元测试覆盖率 | JaCoCo（阈值 60%+） |
| 静态代码分析 | SonarQube、SpotBugs、PMD |
| 代码规范 | Checkstyle |
| 依赖漏洞 | OWASP Dependency-Check、`mvn dependency-check:check`、Trivy |
| 镜像扫描 | Trivy、Grype |
| 许可证 | license-maven-plugin |

```xml
<plugin><groupId>org.jacoco</groupId><artifactId>jacoco-maven-plugin</artifactId>
  <configuration><rules><rule><element>BUNDLE</element>
    <limits><limit><counter>LINE</counter><value>COVEREDRATIO</value>
      <minimum>0.60</minimum></limit></limits></rule></rules></configuration>
  <executions><execution><id>check</id><goals><goal>check</goal></goals></execution></executions>
</plugin>
```

## 8. 发布策略

| 策略 | 做法 | 适用 |
|---|---|---|
| 重建（Recreate） | 先停后启 | 非关键、可中断 |
| 滚动更新 | 逐批替换 | **默认** |
| 蓝绿 | 两套环境切换 | 回滚极快，资源翻倍 |
| 金丝雀 | 5% → 20% → 100% | 高风险变更 |
| 特性开关 | 代码内控制 | 随时开关 |
| 影子流量 | 复制流量验证 | 大改造 |

K8s 金丝雀可结合 Argo Rollouts / Flagger（自动根据指标回滚）。

## 9. Git 分支模型

| 模型 | 说明 |
|---|---|
| Git Flow | main/develop/feature/release/hotfix（重） |
| GitHub Flow | main + feature 分支 + PR（轻，推荐） |
| Trunk Based | 主干开发，频繁合并（需要强 CI 与特性开关） |

版本号：语义化 `MAJOR.MINOR.PATCH`，镜像 tag 用 git commit sha（可追溯）+ 语义版本。

## 10. 实践建议

- 构建**可重复**（依赖锁版本、固定基础镜像 digest）。
- 一次构建，多环境部署（**不要**每个环境重新编译）。
- 制品（镜像）不可变，配置（ConfigMap/Nacos）可变。
- 部署要**幂等**（`kubectl apply`）。
- 流水线要快（< 10 分钟），否则没人愿意跑。
- 失败要**可见**（通知到群/人）。
- 生产发布需要审批 + 灰度 + 可回滚。
- 数据库变更用 Flyway/Liquibase，**纳入流水线**并与应用发布解耦（先加字段后删字段）。

## 11. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| 构建慢 | 每次全量下载 | 依赖缓存、分层镜像、并行 job |
| 本地能过 CI 挂 | 环境差异 | 容器化构建，统一 JDK/Maven 版本 |
| 密钥泄露 | 硬编码进仓库 | 用 secrets/Secret 管理 |
| 测试不稳定（flaky） | 偶发失败 | 隔离、mock 外部依赖、重试或标记 |
| 无回滚方案 | 出事只能硬扛 | 保留上一版本镜像，`rollback` 一键 |
| 直接推生产 | 事故 | 审批 + 灰度 |
| 数据库迁移失败 | 发布中断 | 迁移脚本向后兼容、独立执行 |

## 12. 面试速答

- **CI 与 CD 区别**：CI 是频繁集成（自动构建+测试），CD 是自动交付/部署到环境。
- **流水线包含哪些阶段**：检出 → 编译 → 单测 → 静态扫描 → 打包镜像 → 推仓库 → 部署 → 验证 → 通知。
- **质量门禁**：测试覆盖率、SonarQube、依赖漏洞扫描、镜像扫描。
- **发布策略**：滚动更新（默认）、蓝绿（快速回滚）、金丝雀（风险可控）。
- **为什么一次构建多环境部署**：保证部署到生产的制品与测试通过的是同一个，避免环境差异引入问题。
