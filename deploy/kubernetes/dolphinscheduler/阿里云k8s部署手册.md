# DolphinScheduler 阿里云 ACK 部署文档

本文档用于将当前 Helm Chart 部署到阿里云 Kubernetes 集群。部署时继续使用之前已经推送到百度云 CCR 的 DolphinScheduler 镜像，配置入口为：

```bash
deploy/kubernetes/dolphinscheduler/aliyun-k8s-values.yaml
```

## 1. 部署目标

- Kubernetes 集群：阿里云 ACK
- 部署方式：Helm
- DolphinScheduler 版本：`3.4.1`
- 镜像仓库：继续使用百度云 CCR
- 数据库：外部阿里云 RDS PostgreSQL
- 注册中心：Chart 内置 ZooKeeper，3 副本
- 共享目录：ACK 内可用的 `ReadWriteMany` 存储，用于 `/data/dolphinscheduler`

## 2. 当前 values 关键配置

`aliyun-k8s-values.yaml` 中已经指定百度云 CCR 镜像：

```yaml
image:
  registry: ccr-2owfeef4-pub.cnc.bj.baidubce.com/scheduler
  tag: 3.4.1
  pullPolicy: Always
  pullSecret: ""
  master: dolphinscheduler-master-with-plugin
  worker: dolphinscheduler-worker-with-plugin-py3
  api: dolphinscheduler-api-with-plugin
  alert: dolphinscheduler-alert-server-with-plugin
  tools: dolphinscheduler-tools
```

依赖镜像也继续使用同一个百度云 CCR 仓库，例如 `busybox`、`zookeeper`、`postgresql`、`mysql`、`minio`。

数据库使用外部 PostgreSQL：

```yaml
postgresql:
  enabled: false

externalDatabase:
  enabled: true
  type: postgresql
  host: pgm-2ze018200m73425d.pg.rds.aliyuncs.com
  port: "5432"
  username: dolphin_user
  database: dolphinscheduler
  params: currentSchema=public
  driverClassName: org.postgresql.Driver
```

注意：数据库密码请以实际部署文件或 Secret 为准，不要在共享文档、工单、IM 中明文传播。

共享存储当前配置为：

```yaml
common:
  sharedStoragePersistence:
    enabled: true
    mountPath: /data/dolphinscheduler
    accessModes:
      - ReadWriteMany
    storageClassName: alicloud-nas-extreme-rwx
    storage: 50Gi
```

在阿里云 ACK 上需要保证存在同名 StorageClass。当前使用的 `alicloud-nas-extreme-rwx` 是阿里云 NAS Extreme RWX 存储类，支持多 Pod 读写。

## 3. 前置条件

本地或跳板机需要安装：

```bash
kubectl version --client
helm version
```

确认 kubeconfig 中存在目标 ACK context。本次部署固定使用 `aliyun-common-service`：

```bash
export KUBE_CONTEXT=aliyun-common-service

kubectl config get-contexts
kubectl --context ${KUBE_CONTEXT} cluster-info
kubectl --context ${KUBE_CONTEXT} get nodes
```

建议使用独立命名空间，下面示例统一使用 `bigdata`：

```bash
export KUBE_CONTEXT=aliyun-common-service
export NAMESPACE=bigdata
export RELEASE=dolphinscheduler
export CHART=deploy/kubernetes/dolphinscheduler/dolphinscheduler-helm
export VALUES=deploy/kubernetes/dolphinscheduler/aliyun-k8s-values.yaml

kubectl --context ${KUBE_CONTEXT} create namespace ${NAMESPACE} --dry-run=client -o yaml \
  | kubectl --context ${KUBE_CONTEXT} apply -f -
```

## 4. 镜像拉取说明

当前百度云 CCR 镜像无需认证拉取，因此不需要创建 `imagePullSecrets`。`aliyun-k8s-values.yaml` 中保持：

```yaml
image:
  pullSecret: ""
```

子 Chart 依赖镜像，例如 ZooKeeper、PostgreSQL、MySQL、MinIO，也不配置 `pullSecrets`。

## 5. 准备共享存储

DolphinScheduler 的 API、Master、Worker 会挂载 `/data/dolphinscheduler`，用于放置 Hadoop、Spark、Hive、Flink、DataX 等客户端和配置。该卷必须支持多 Pod 读写。

先查看 ACK 集群已有 StorageClass：

```bash
kubectl --context ${KUBE_CONTEXT} get storageclass
```

本次部署使用的 StorageClass 为：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: alicloud-nas-extreme-rwx
provisioner: nasplugin.csi.alibabacloud.com
parameters:
  archiveOnDelete: "true"
  server: 011rv6vpjoxsx3d1gc1-sbiq.cn-beijing.extreme.nas.aliyuncs.com:/k8s
  volumeAs: subpath
mountOptions:
  - nolock,tcp,noresvport
  - vers=3
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: Immediate
```

如果目标集群中还没有该 StorageClass，需要先创建；如果已经存在，直接确认名称即可：

```bash
kubectl --context ${KUBE_CONTEXT} get storageclass alicloud-nas-extreme-rwx
```

安装后确认 PVC 能正常 Bound：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} get pvc
```

## 6. 准备 RDS PostgreSQL

确认 RDS PostgreSQL 满足以下条件：

- ACK 节点网络可以访问 RDS 地址和 `5432` 端口。
- RDS 白名单已放行 ACK 节点网段或 Pod 出口网段。
- 数据库 `dolphinscheduler` 已存在。
- 用户 `dolphin_user` 对数据库和 `public` schema 有建表、改表、读写权限。

参考 SQL：

```sql
CREATE DATABASE dolphinscheduler;
CREATE USER dolphin_user WITH PASSWORD '<数据库密码>';
GRANT ALL PRIVILEGES ON DATABASE dolphinscheduler TO dolphin_user;

\c dolphinscheduler
GRANT ALL ON SCHEMA public TO dolphin_user;
ALTER SCHEMA public OWNER TO dolphin_user;
```

从集群内测试连通性：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} run pg-check --rm -it --restart=Never \
  --image=ccr-2owfeef4-pub.cnc.bj.baidubce.com/scheduler/busybox:1.30.1 \
  -- sh
```

进入容器后执行：

```sh
nc -vz pgm-2ze018200m73425d.pg.rds.aliyuncs.com 5432
```

如果 busybox 镜像无 `nc`，可换用带 PostgreSQL 客户端的临时镜像测试。

## 7. 部署前检查 values

部署前重点检查这些字段：

```bash
grep -nE 'registry:|tag:|pullSecret:|externalDatabase:|host:|storageClassName:|RESOURCE_STORAGE_TYPE|sharedStoragePersistence:' ${VALUES}
```

确认：

- `image.registry` 仍为百度云 CCR。
- `image.tag` 为预期版本。
- `image.pullSecret` 为空，部署时不引用镜像拉取 Secret。
- `externalDatabase.host`、用户名、数据库、密码正确。
- `common.sharedStoragePersistence.storageClassName` 在 ACK 中存在且支持 RWX。
- 当前资源存储为 `LOCAL`，如果生产要使用 OSS，需要同步调整 `conf.common.resource.storage.type`、`common.configmap.RESOURCE_STORAGE_TYPE` 及 OSS 参数。

先渲染 Chart，提前发现 YAML 或模板错误：

```bash
helm template ${RELEASE} ${CHART} \
  --kube-context ${KUBE_CONTEXT} \
  -n ${NAMESPACE} \
  -f ${VALUES} > /tmp/dolphinscheduler-rendered.yaml
```

当前 values 渲染时可能出现 Helm 类型覆盖警告，例如 `worker.initContainers` 从默认 `{}` 覆盖为列表、`ingress.tls` 从默认对象覆盖为 `false`。只要 `helm template` 最终退出码为 0，一般不影响本次部署；如后续要启用 Ingress TLS，建议把 `ingress.tls` 按 Chart 默认结构改回对象配置。

## 8. 安装

执行 Helm 安装：

```bash
helm upgrade --install ${RELEASE} ${CHART} \
  --kube-context ${KUBE_CONTEXT} \
  -n ${NAMESPACE} \
  -f ${VALUES} \
  --timeout 15m \
  --wait
```

Chart 会在安装、升级、回滚后通过 Helm hook 创建数据库初始化 Job：

```text
${RELEASE}-dolphinscheduler-db-init-job
```

该 Job 使用 `dolphinscheduler-tools:3.4.1` 镜像执行：

```bash
tools/bin/upgrade-schema.sh
```

## 9. 部署验证

查看 Helm 状态：

```bash
helm --kube-context ${KUBE_CONTEXT} -n ${NAMESPACE} status ${RELEASE}
```

查看 Pod：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} get pods -o wide
```

期望看到：

- `dolphinscheduler-master-*` 为 Running
- `dolphinscheduler-worker-*` 为 Running
- `dolphinscheduler-api-*` 为 Running
- `dolphinscheduler-alert-*` 为 Running
- `dolphinscheduler-zookeeper-*` 为 Running
- `dolphinscheduler-db-init-job-*` 为 Completed

查看初始化 Job：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} get job
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} logs job/${RELEASE}-dolphinscheduler-db-init-job
```

查看服务：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} get svc
```

当前 values 中 API Service 是 `ClusterIP`。本地访问 UI：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} port-forward svc/${RELEASE}-dolphinscheduler-api 12345:12345
```

浏览器打开：

```text
http://127.0.0.1:12345/dolphinscheduler
```

默认登录账号通常为：

```text
admin / dolphinscheduler123
```

如需通过公网或内网 SLB 访问，可以将 `api.service.type` 改为 `LoadBalancer`，或启用 `ingress.enabled` 并配置 ACK Ingress。

## 10. 上传运行依赖到共享目录

当前 values 将客户端路径指向 `/data/dolphinscheduler/soft`：

```yaml
HADOOP_HOME: /data/dolphinscheduler/soft/hadoop
HADOOP_CONF_DIR: /data/dolphinscheduler/soft/hadoop/etc/hadoop
SPARK_HOME: /data/dolphinscheduler/soft/spark
HIVE_HOME: /data/dolphinscheduler/soft/hive
FLINK_HOME: /data/dolphinscheduler/soft/flink
DATAX_LAUNCHER: /data/dolphinscheduler/soft/datax/bin/datax.py
```

如需运行 Spark、Hive、Flink、DataX 等任务，需要提前把对应客户端和配置上传到共享存储。可以临时进入 Worker Pod 检查挂载目录：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} exec -it statefulset/${RELEASE}-dolphinscheduler-worker -- sh
ls -lah /data/dolphinscheduler
```

建议目录结构：

```text
/data/dolphinscheduler/
  soft/
    hadoop/
    spark/
    hive/
    flink/
    datax/
```

## 11. 升级

修改 `aliyun-k8s-values.yaml` 后执行：

```bash
helm upgrade ${RELEASE} ${CHART} \
  --kube-context ${KUBE_CONTEXT} \
  -n ${NAMESPACE} \
  -f ${VALUES} \
  --timeout 15m \
  --wait
```

升级后确认：

```bash
helm --kube-context ${KUBE_CONTEXT} -n ${NAMESPACE} history ${RELEASE}
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} get pods
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} get job
```

## 12. 回滚

查看历史版本：

```bash
helm --kube-context ${KUBE_CONTEXT} -n ${NAMESPACE} history ${RELEASE}
```

回滚到指定 revision：

```bash
helm --kube-context ${KUBE_CONTEXT} -n ${NAMESPACE} rollback ${RELEASE} <REVISION> --timeout 15m --wait
```

注意：回滚 Helm 资源不等于回滚数据库 schema。生产环境升级前建议先备份 RDS。

## 13. 卸载

```bash
helm --kube-context ${KUBE_CONTEXT} -n ${NAMESPACE} uninstall ${RELEASE}
```

PVC、PV、RDS 数据库通常不会自动删除。确认无用后再清理：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} get pvc
```

## 14. 常见问题

### ImagePullBackOff

当前部署不使用镜像拉取 Secret。出现 `ImagePullBackOff` 时，优先检查 ACK 节点到百度云 CCR 的网络连通性、镜像名称和 tag 是否存在：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} describe pod <pod-name>
```

### PVC Pending

检查 StorageClass 是否存在、是否支持 `ReadWriteMany`：

```bash
kubectl --context ${KUBE_CONTEXT} get storageclass
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} describe pvc <pvc-name>
```

### 数据库初始化失败

查看 Job 日志：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} logs job/${RELEASE}-dolphinscheduler-db-init-job
```

重点检查 RDS 白名单、数据库权限、密码、`externalDatabase.params`。

### Pod 一直 Init

查看 init container 日志和事件：

```bash
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} describe pod <pod-name>
kubectl --context ${KUBE_CONTEXT} -n ${NAMESPACE} logs <pod-name> -c <init-container-name>
```

### UI 无法访问

当前默认是 `ClusterIP`，需要使用 port-forward 访问。如果要通过 SLB/Ingress 暴露，调整 `api.service.type` 或 `ingress` 配置后重新 `helm upgrade`。

## 15. 生产建议

- RDS 密码、AccessKey 等敏感信息不要长期明文放在 values 文件中，后续建议改造为引用已有 Kubernetes Secret。
- ZooKeeper 已开启 3 副本和持久化，但当前 `storageClass: "-"` 会禁用动态供给；生产环境建议配置 ACK 可用的持久化 StorageClass。
- 为 Master、Worker、API 设置合理的 `nodeSelector`、`affinity`、`tolerations`，避免全部调度到单节点。
- 根据任务规模调整 `master.replicas`、`worker.replicas` 和 JVM 参数。
- 开启公网访问前，优先使用内网 SLB、Ingress 白名单、HTTPS 和认证加固。
- 每次升级前备份 RDS，并保存当前 `helm history` 和 values。
