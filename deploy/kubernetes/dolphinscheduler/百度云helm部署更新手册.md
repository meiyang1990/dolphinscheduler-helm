# DolphinScheduler Helm 更新操作手册

本文档记录 `bigdata` 命名空间中 DolphinScheduler Helm 部署的常用更新流程，适用于调整 worker 资源、并发、实例数，以及更新后验证和回滚。

## 基本信息

- Release 名称: `dolphinscheduler`
- Namespace: `bigdata`
- Chart 路径: `deploy/kubernetes/dolphinscheduler/dolphinscheduler-helm`
- Values 文件: `deploy/kubernetes/dolphinscheduler/baidu-k8s-dolphinscheduler.yaml`
- Worker StatefulSet: `dolphinscheduler-worker`

## 更新前检查

确认当前 release 和 pod 状态:

```bash
helm list -n bigdata | grep dolphinscheduler
helm history dolphinscheduler -n bigdata
kubectl get pods -n bigdata | grep dolphinscheduler
kubectl get sts dolphinscheduler-worker -n bigdata
```

查看 worker 当前资源和并发:

```bash
kubectl get sts dolphinscheduler-worker -n bigdata \
  -o jsonpath='{.spec.replicas}{" replicas\n"}{.spec.template.spec.containers[0].resources}{"\n"}{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}' \
  | grep -E 'replicas|memory|cpu|WORKER_EXEC_THREADS'
```

如果 worker 出现 OOM，先确认退出原因:

```bash
kubectl describe pod dolphinscheduler-worker-0 -n bigdata
kubectl logs dolphinscheduler-worker-0 -n bigdata --previous --tail=200
kubectl top pod -n bigdata | grep dolphinscheduler-worker
```

## 修改 Values

编辑:

```bash
deploy/kubernetes/dolphinscheduler/baidu-k8s-dolphinscheduler.yaml
```

示例 worker 配置:

```yaml
worker:
  replicas: "2"

  resources:
    limits:
      memory: "16Gi"
      cpu: "4"
    requests:
      memory: "8Gi"
      cpu: "2"

  env:
    WORKER_EXEC_THREADS: "5"
```

说明:

- `WORKER_EXEC_THREADS` 控制单个 worker 同时执行的任务数。
- pod 内存包含 worker JVM、Python/Shell/Spark 提交进程、native/offheap、线程栈等全部内存。
- 如果 Spark 任务很多或提交端占用较大，优先降低并发，再考虑增大内存。

## 执行 Helm 更新

在仓库根目录执行:

```bash
helm upgrade dolphinscheduler \
  deploy/kubernetes/dolphinscheduler/dolphinscheduler-helm \
  -n bigdata \
  -f deploy/kubernetes/dolphinscheduler/baidu-k8s-dolphinscheduler.yaml
```

常见 warning:

```text
cannot overwrite table with non table
```

如果 release 最终显示 `STATUS: deployed`、`DESCRIPTION: Upgrade complete`，一般可继续验证。

## 更新后验证

等待 worker 滚动更新完成:

```bash
kubectl rollout status sts/dolphinscheduler-worker -n bigdata --timeout=180s
```

确认 StatefulSet 模板:

```bash
kubectl get sts dolphinscheduler-worker -n bigdata \
  -o jsonpath='{.spec.replicas}{" replicas\n"}{.spec.template.spec.containers[0].resources}{"\n"}{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}' \
  | grep -E 'replicas|memory|cpu|WORKER_EXEC_THREADS'
```

确认实际 pod:

```bash
kubectl get pods -n bigdata | grep dolphinscheduler-worker
kubectl top pod -n bigdata | grep dolphinscheduler-worker
kubectl describe pod dolphinscheduler-worker-0 -n bigdata | sed -n '/Containers:/,/Conditions:/p'
```

重点检查:

- pod 是否 `1/1 Running`
- `Restart Count` 是否持续增加
- `Last State` 是否仍为 `OOMKilled`
- 内存是否接近 limit

## 停止当前运行中的工作流实例

优先使用 DolphinScheduler API 停止，不建议直接改数据库。

登录 API pod:

```bash
API_POD=$(kubectl get pod -n bigdata -l app.kubernetes.io/component=api -o jsonpath='{.items[0].metadata.name}')
```

获取 `sessionId`:

```bash
SESSION_ID=$(kubectl exec -n bigdata "$API_POD" -- curl -sS -X POST \
  'http://localhost:12345/dolphinscheduler/login' \
  --data-urlencode 'userName=<admin-user>' \
  --data-urlencode 'userPassword=<admin-password>' \
  | sed -n 's/.*"sessionId":"\([^"]*\)".*/\1/p')
```

查询项目:

```bash
kubectl exec -n bigdata "$API_POD" -- curl -sS \
  -H "sessionId: $SESSION_ID" \
  'http://localhost:12345/dolphinscheduler/projects/list'
```

查询某项目运行中实例:

```bash
PROJECT_CODE=<project-code>

kubectl exec -n bigdata "$API_POD" -- curl -sS \
  -H "sessionId: $SESSION_ID" \
  "http://localhost:12345/dolphinscheduler/projects/${PROJECT_CODE}/workflow-instances?stateType=RUNNING_EXECUTION&pageNo=1&pageSize=50"
```

批量停止实例:

```bash
WORKFLOW_INSTANCE_IDS=<id1,id2,id3>

kubectl exec -n bigdata "$API_POD" -- curl -sS \
  -H "sessionId: $SESSION_ID" \
  -X POST \
  "http://localhost:12345/dolphinscheduler/projects/${PROJECT_CODE}/executors/batch-execute?workflowInstanceIds=${WORKFLOW_INSTANCE_IDS}&executeType=STOP"
```

停止后复查:

```bash
kubectl exec -n bigdata "$API_POD" -- curl -sS \
  -H "sessionId: $SESSION_ID" \
  "http://localhost:12345/dolphinscheduler/projects/${PROJECT_CODE}/workflow-instances?stateType=RUNNING_EXECUTION&pageNo=1&pageSize=1"

kubectl exec -n bigdata "$API_POD" -- curl -sS \
  -H "sessionId: $SESSION_ID" \
  "http://localhost:12345/dolphinscheduler/projects/${PROJECT_CODE}/workflow-instances?stateType=READY_STOP&pageNo=1&pageSize=1"
```

说明:

- `RUNNING_EXECUTION=0` 表示没有正在运行的工作流实例。
- `READY_STOP` 是停止命令下发后的过渡态，master/worker 处理完成后会变成 `STOP`。
- 如果定时调度仍在线，新的实例可能继续产生，需要先暂停相关 schedule。

## 回滚

查看历史版本:

```bash
helm history dolphinscheduler -n bigdata
```

回滚到指定 revision:

```bash
helm rollback dolphinscheduler <revision> -n bigdata
kubectl rollout status sts/dolphinscheduler-worker -n bigdata --timeout=180s
```

回滚后再次确认:

```bash
kubectl get pods -n bigdata | grep dolphinscheduler-worker
kubectl get sts dolphinscheduler-worker -n bigdata \
  -o jsonpath='{.spec.replicas}{" replicas\n"}{.spec.template.spec.containers[0].resources}{"\n"}{range .spec.template.spec.containers[0].env[*]}{.name}={.value}{"\n"}{end}' \
  | grep -E 'replicas|memory|cpu|WORKER_EXEC_THREADS'
```

## 常见问题

### 重启 master 能否丢弃运行任务

不建议。DolphinScheduler 的实例状态存储在数据库中，master 重启后可能触发 failover/recover，运行中的任务可能被重新接管。需要停止实例时，优先通过 UI 或 API 执行 `STOP`。

### worker 调大到 16Gi 仍然 OOM

常见原因是 Spark/Python/Shell 子进程内存也算入 worker pod。处理顺序建议:

1. 降低 `WORKER_EXEC_THREADS`，例如从 `5` 降到 `2`。
2. 增加 worker 副本数，让任务分摊到多个 pod。
3. 增大 worker `limits.memory`，例如 `24Gi` 或 `32Gi`。
4. 检查 Spark 任务参数，例如 `driverMemory`、`executorMemory`、`numExecutors`。
5. 暂停高频 schedule，避免新实例持续堆积。
