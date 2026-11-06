# 64 · Kubernetes 部署

## 1. 核心对象

| 对象 | 作用 |
|---|---|
| Pod | 最小调度单元（1+ 容器） |
| Deployment | 无状态应用（副本、滚动更新） |
| StatefulSet | 有状态（稳定网络标识、持久卷） |
| Service | 稳定访问入口（ClusterIP/NodePort/LoadBalancer） |
| Ingress | HTTP 路由（7 层） |
| ConfigMap | 配置 |
| Secret | 敏感配置（base64） |
| PVC/PV | 持久化存储 |
| HPA | 自动扩缩容 |
| Job/CronJob | 任务 |
| Namespace | 资源隔离 |

## 2. 最小 Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels: {app: order}
  strategy:
    type: RollingUpdate
    rollingUpdate: {maxSurge: 1, maxUnavailable: 0}
  template:
    metadata:
      labels: {app: order}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
    spec:
      terminationGracePeriodSeconds: 60      # ★ 要大于优雅停机时间
      containers:
        - name: order
          image: registry.example.com/order:1.2.3
          ports: [{containerPort: 8080}]
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: JAVA_OPTS
              value: "-XX:MaxRAMPercentage=70 -XX:+ExitOnOutOfMemoryError"
          envFrom:
            - configMapRef: {name: order-config}
            - secretRef: {name: order-secret}
          resources:
            requests: {cpu: "500m", memory: "1Gi"}
            limits:   {cpu: "2",    memory: "2Gi"}
          startupProbe:                       # ★ 慢启动用 startupProbe
            httpGet: {path: /actuator/health/liveness, port: 8080}
            failureThreshold: 30
            periodSeconds: 5
          readinessProbe:                     # 决定能否接流量
            httpGet: {path: /actuator/health/readiness, port: 8080}
            periodSeconds: 5
            failureThreshold: 3
          livenessProbe:                      # 决定是否重启
            httpGet: {path: /actuator/health/liveness, port: 8080}
            periodSeconds: 10
            failureThreshold: 3
            initialDelaySeconds: 30
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 15"]   # ★ 等待注册中心/网关摘流
---
apiVersion: v1
kind: Service
metadata: {name: order-service, namespace: prod}
spec:
  selector: {app: order}
  ports: [{port: 8080, targetPort: 8080}]
  type: ClusterIP
```

## 3. 探针要点

| 探针 | 失败后果 | 用途 |
|---|---|---|
| `startupProbe` | 重启 | 保护慢启动（成功后才交给另两个） |
| `readinessProbe` | **摘除流量**（不重启） | 依赖未就绪、正在关闭 |
| `livenessProbe` | **重启容器** | 死锁/假死 |

> 常见错误：liveness 探测依赖外部服务（DB 挂了就无限重启）→ liveness 只检查**自身**是否假死。

Boot 侧：
```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true       # 生成 /actuator/health/liveness 和 /readiness
  health:
    livenessstate: {enabled: true}
    readinessstate: {enabled: true}
```
`AvailabilityChangeEvent` 可让应用在关闭时主动置 readiness DOWN。

## 4. 配置与密钥

```bash
kubectl create configmap order-config --from-file=application-prod.yml
kubectl create secret generic order-secret \
  --from-literal=SPRING_DATASOURCE_PASSWORD=xxx
```

```yaml
# 挂载文件方式
containers:
  - volumeMounts:
      - name: cfg
        mountPath: /config
volumes:
  - name: cfg
    configMap: {name: order-config}
```
启动参数：`--spring.config.additional-location=file:/config/`

> 密钥管理进阶：Sealed Secrets、Vault、External Secrets Operator。

## 5. 优雅停机（关键）

时序：
```
1 kubectl delete pod → 发送 SIGTERM，同时从 Service Endpoint 移除（异步）
2 preStop: sleep 15        ← 等待 Endpoint 移除与调用方感知
3 Spring 收到 SIGTERM → shutdown=graceful，等待在途请求完成（最多 30s）
4 容器退出；超过 terminationGracePeriodSeconds 则 SIGKILL
```

要点：
- `terminationGracePeriodSeconds` > preStop sleep + 优雅停机超时。
- Nacos/Eureka 场景：preStop 时间要大于客户端缓存刷新周期（或主动反注册）。
- 处理 SIGTERM（不要被 shell 吞掉，用 exec 形式 ENTRYPOINT）。

## 6. 扩缩容

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: order-hpa}
spec:
  scaleTargetRef: {apiVersion: apps/v1, kind: Deployment, name: order-service}
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource: {name: cpu, target: {type: Utilization, averageUtilization: 70}}
    - type: Resource
      resource: {name: memory, target: {type: Utilization, averageUtilization: 80}}
  behavior:
    scaleDown: {stabilizationWindowSeconds: 300}
```
- 自定义指标（QPS）需 Prometheus Adapter。
- VPA：调整 requests/limits。
- KEDA：基于事件（Kafka lag）扩缩容。

## 7. Helm

```
mychart/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── configmap.yaml
```

```yaml
# values.yaml
replicaCount: 3
image:
  repository: registry.example.com/order
  tag: 1.2.3
resources:
  requests: {cpu: 500m, memory: 1Gi}
```

```bash
helm install order ./mychart -f values-prod.yaml -n prod
helm upgrade order ./mychart --set image.tag=1.2.4
helm rollback order 1
```

## 8. Spring Cloud Kubernetes（可选）

```xml
<dependency><groupId>org.springframework.cloud</groupId>
           <artifactId>spring-cloud-starter-kubernetes-fabric8</artifactId></dependency>
```
- 用 K8s Service 做服务发现（无需 Nacos/Eureka）。
- ConfigMap/Secret 作为配置源，支持**热更新**（`spring.cloud.kubernetes.reload.enabled=true`）。
- 注意：生产更推荐普通 K8s 原生方式（DNS + ConfigMap），少一层依赖。

## 9. 常用命令

```bash
kubectl get pods -n prod -o wide
kubectl logs -f deploy/order-service --tail=100 -c order
kubectl logs -p <pod>              # 上一个容器的日志（崩溃排查）
kubectl describe pod <pod>         # 事件（调度失败、探针失败）
kubectl exec -it <pod> -- sh
kubectl port-forward svc/order-service 8080:8080
kubectl rollout status deploy/order-service
kubectl rollout undo deploy/order-service
kubectl scale deploy/order-service --replicas=5
kubectl top pods -n prod
kubectl apply -f k8s/ && kubectl delete -f k8s/
kubectl get events --sort-by=.lastTimestamp -n prod
```

## 10. 常见坑

| 坑 | 现象 | 解决 |
|---|---|---|
| CrashLoopBackOff | 反复重启 | 看日志与 `describe`；常见：配置缺失、OOM、探针失败 |
| OOMKilled | 被杀 | 调大 limit 或减小 `MaxRAMPercentage` |
| ImagePullBackOff | 拉不到镜像 | 检查仓库地址、imagePullSecrets |
| 发布时 502 | 流量打到已关闭 Pod | preStop sleep + readiness + graceful shutdown |
| 探针失败 | 不断重启 | liveness 不依赖外部；调整 initialDelay |
| 服务名解析失败 | UnknownHost | 用 Service 名（同 namespace）；跨 ns 用 FQDN |
| 配置更新不生效 | 需重启 | ConfigMap 挂载文件 + reload，或滚动重启 |
| CPU throttling | 延迟抖动 | 合理设置 limit（或只设 requests） |
| 滚动更新中断 | 旧版本被杀太快 | `maxUnavailable: 0` + PDB |

## 11. 面试速答

- **Pod 与 Deployment**：Pod 是最小运行单元；Deployment 管理副本与滚动更新。
- **三种探针**：startup（保护启动）、readiness（能否接流量，失败摘流量）、liveness（是否假死，失败重启）。
- **优雅停机怎么做**：preStop sleep + `terminationGracePeriodSeconds` + Spring `server.shutdown=graceful`，保证在途请求完成且流量已摘除。
- **ConfigMap 与 Secret**：前者放普通配置，后者放敏感信息（base64，非加密，生产用 Vault/Sealed Secrets）。
- **HPA 依据什么**：默认 CPU/内存，可接自定义指标（QPS）；缩容要设稳定窗口避免抖动。
