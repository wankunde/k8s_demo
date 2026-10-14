# 05：PriorityClass 与 Volcano Preempt

本例展示同一 Queue 内的 Job 抢占：低优先级 Job 先占据目标节点，高优先级 Job 后提交。两个 Job 都请求 `20 CPU`，并通过 `volcano-preempt-demo=ready` 只允许落到同一个节点。目标节点若有 28 CPU，则两个 Job 可以分别运行，但不能同时运行；Volcano 必须驱逐低优先级 victim，才能让高优先级 preemptor 运行。

流程：

```text
low(-100) Running -> high(100) Pending -> preempt action 选择 low Pod
                  -> low Pod Evicted -> high Running
                  -> high Completed -> low Job 重启并再次 Running
```

## 1. UAT 当前为何不会抢占

2026-10-13 只读检查确认：UAT 的 scheduler 镜像为 Volcano `v1.15.0`，`volcano-system/volcano-scheduler-configmap` 只有 `actions: "allocate"`。`priority` 插件已经存在，`gang.enablePreemptable` 为 `false`，但调度流水线没有 `preempt` action，所以高优先级只影响排序，不会驱逐已经运行的低优先级 Pod。

开启本例需要同时满足：

1. `actions` 包含 `preempt`，本例使用 `"allocate, preempt"`。
2. `priority` 插件启用 PreemptableFn；未显式配置时 v1.15 默认是 `true`，示例仍明确写出 `enablePreemptable: true`。
3. preemptor 与 victim 属于同一个 Queue。跨 Queue 借用资源的回收属于 `reclaim`，不是本例的 `preempt`。
4. preemptor 的 Job 优先级高于 victim，且 `preemptionPolicy` 不能为 `Never`。
5. preemptor 确实因资源不足无法调度；若还有空闲节点，它会直接调度，不会产生 victim。

本例保持 `gang.enablePreemptable: false`。如果设为 `true`，Gang 插件会保护处于 `minAvailable` 下限的 Job；victim Job 只有一个 Pod 且 `minAvailable=1`，Gang 与 Priority 两个插件的 victim 结果取交集后将没有可驱逐对象。

## 2. 开启 preempt

修改 scheduler ConfigMap 会改变整个 UAT 集群的调度行为。启用后，其他 Queue 内存在优先级差和资源争用时，也可能发生抢占。先备份当前配置并核对差异：

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

"$K" uat get configmap volcano-scheduler-configmap -n volcano-system -o yaml \
  > /tmp/volcano-scheduler-configmap.before-preempt.yaml

diff -u \
  <("$K" uat get configmap volcano-scheduler-configmap -n volcano-system -o jsonpath='{.data.volcano-scheduler\.conf}') \
  <(ruby -r yaml -e 'print YAML.load_file(ARGV[0])["data"]["volcano-scheduler.conf"]' \
    /Users/wankun/ws/kunwan/k8s_demo/volcano/05-preempt-scheduler-config.yaml)
```

预期有效差异只有 `actions` 增加 `preempt`，以及 `priority` 显式增加 `enablePreemptable: true`；其余已有插件及参数保持不变。纯文本 diff 还会显示一处中文注释行尾空格被规范化，它不影响配置语义。确认差异后手工应用：

```bash
"$K" uat apply -f - \
  < /Users/wankun/ws/kunwan/k8s_demo/volcano/05-preempt-scheduler-config.yaml
```

Volcano v1.15 会监听挂载的配置文件并热加载，不要求重启 scheduler。检查配置文件和加载日志：

```bash
"$K" uat exec -n volcano-system deploy/volcano-scheduler -- \
  cat /volcano.scheduler/volcano-scheduler.conf

"$K" uat logs -n volcano-system deploy/volcano-scheduler --since=5m \
  | rg 'Finished loading scheduler config|actions=|invalid|Invalid|failed|Failed'
```

日志应包含类似 `actions=[allocate preempt]`，且没有 invalid/failed 配置错误。如果配置错误，v1.15 会继续使用上一次有效配置，不能只凭 ConfigMap 已更新判断启用成功。

## 3. 创建专用 Queue 和优先级

本例用专用 Queue 限定 victim 范围，避免把 `default` Queue 中的其他低优先级 Volcano Job 当成 victim：

```bash
"$K" uat apply -f - \
  < /Users/wankun/ws/kunwan/k8s_demo/volcano/05-preempt-prerequisites.yaml

"$K" uat get queue volcano-learn-05-preempt
"$K" uat get priorityclass volcano-learn-05-low volcano-learn-05-high \
  -o 'custom-columns=NAME:.metadata.name,VALUE:.value,POLICY:.preemptionPolicy'
```

预期值：low 为 `-100`，high 为 `100`，两者 policy 都是 `PreemptLowerPriority`。

## 4. 选择并隔离一个实验节点

示例 CPU 请求是按 28 CPU 节点设计的。2026-10-13 只读检查 `10.159.192.50`：Ready、allocatable CPU 为 28，当时已有 Pod 的 CPU requests 合计约 `450m`。运行前必须重新核对；不要在业务繁忙节点直接做抢占实验。

```bash
TARGET_NODE=10.159.192.50

"$K" uat get node "$TARGET_NODE" \
  -o 'custom-columns=NAME:.metadata.name,READY:.status.conditions[?(@.type=="Ready")].status,CPU:.status.allocatable.cpu'

"$K" uat describe node "$TARGET_NODE"
"$K" uat get pods -A --field-selector "spec.nodeName=$TARGET_NODE" \
  -o 'custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,PHASE:.status.phase,CPU:.spec.containers[*].resources.requests.cpu,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName'

# 只有这个标签命中的节点才是本例候选节点。
"$K" uat label node "$TARGET_NODE" volcano-preempt-demo=ready
"$K" uat get nodes -l volcano-preempt-demo=ready
```

必须保证标签只匹配一个节点、节点至少能容纳单个 `20 CPU` Job，并确认节点上没有这个专用 Queue 的其他实验 Job。

## 5. 先启动 victim，再启动 preemptor

先提交低优先级 Job，并等待其 Pod 已经 `Running`。如果 low 尚未运行就提交 high，只能证明优先级排序，不能证明抢占。

```bash
"$K" uat apply -f - \
  < /Users/wankun/ws/kunwan/k8s_demo/volcano/05-preempt-low-job.yaml

"$K" uat get pods -n infra-demo-prod -l app=volcano-learn,lesson=05 -o wide -w
```

另开终端；确认 low Pod 为 `Running` 后再提交 high：

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

"$K" uat apply -f - \
  < /Users/wankun/ws/kunwan/k8s_demo/volcano/05-preempt-high-job.yaml
```

## 6. 观察抢占证据

在 k9s 中选择 `infra-demo-prod`，依次打开 `:vcjob`、`:podgroups.scheduling.volcano.sh`、`:pods`，并用 `/volcano-learn-05` 过滤。victim Pod 会被驱逐，preemptor Pod 随后运行；高优先级 Job 结束后，低优先级 Job 会按 `PodEvicted -> RestartJob` 策略恢复。

命令行观察：

```bash
"$K" uat get jobs.batch.volcano.sh -n infra-demo-prod -l app=volcano-learn,lesson=05 \
  -o 'custom-columns=NAME:.metadata.name,QUEUE:.spec.queue,CLASS:.spec.priorityClassName,PHASE:.status.state.phase'

"$K" uat get pods -n infra-demo-prod -l app=volcano-learn,lesson=05 \
  -o 'custom-columns=NAME:.metadata.name,ROLE:.metadata.labels.role,PHASE:.status.phase,NODE:.spec.nodeName,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName' -w
```

事件和 scheduler 日志是抢占的直接证据：

```bash
"$K" uat get events -n infra-demo-prod \
  --field-selector reason=Evict --sort-by=.lastTimestamp

"$K" uat logs -n volcano-system deploy/volcano-scheduler --since=10m \
  | rg 'Try to preempt Task|Evicting pod|volcano-learn-05'
```

成功标准不是 low 消失或 high 变成 Running 中的任意一个单独现象，而是同一时间窗口中同时看到：

- scheduler 记录选择 low Pod 为 victim，例如 `Try to preempt Task <...low...> for Task <...high...>`；
- low Pod 出现 Evicted/DisruptionTarget 证据；
- high Pod 随后绑定到目标节点并进入 Running；
- high 完成后，low Job 创建的新 Pod 能重新运行。

## 7. 恢复边界

本例未自动修改 UAT。实验后通过备份恢复 ConfigMap 的 `data`，再移除 `volcano-preempt-demo` 节点标签：

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

restore_patch=$(ruby -r yaml -r json -e \
  'doc=YAML.load_file(ARGV[0]); print JSON.generate({"data" => doc.fetch("data")})' \
  /tmp/volcano-scheduler-configmap.before-preempt.yaml)

"$K" uat patch configmap volcano-scheduler-configmap -n volcano-system \
  --type=merge -p "$restore_patch"

"$K" uat label node 10.159.192.50 volcano-preempt-demo-

"$K" uat logs -n volcano-system deploy/volcano-scheduler --since=5m \
  | rg 'Finished loading scheduler config|actions=|invalid|Invalid|failed|Failed'
```

恢复日志中的 action 列表应回到实验前的值。Queue 和 PriorityClass 是集群级对象，不会随 Job TTL 删除；如需移除，由集群管理员按精确名称手工处理。

Job 设置了 `ttlSecondsAfterFinished: 300`，只有 Job 真正完成后才开始 TTL；低优先级 Job 如果一直处于重启或 Pending，不会自动进入 TTL 清理。Queue、PriorityClass 和节点标签也不会随 Job TTL 删除。

## 验证记录

本次使用 UAT 当前 ConfigMap、scheduler Deployment 镜像/参数、PriorityClass 列表和节点容量做了只读核对；并对 Volcano v1.15.0 的 scheduler 配置热加载、preempt action、priority 插件和 Gang victim 交集逻辑做了源码核对。YAML 已做本地解析和字段一致性检查。

专用 Queue、两个 PriorityClass 和两个 Job 的精确名称预检查均返回 NotFound。[remote-k8s-operations 技能](/Users/wankun/.codex/skills/remote-k8s-operations/SKILL.md) 对新对象要求："report that and do not run the operation"，因此本次没有应用共享 scheduler 配置，也没有创建实验对象。真实驱逐、Job 恢复和 TTL 行为仍待按本文步骤运行验证。

本地 `kubectl apply --dry-run=client --validate=false` 仍尝试访问未配置的 `localhost:8080`，因此该检查未完成；这不算清单通过。已通过的本地检查是四个文件的 YAML 解析、嵌套 scheduler 配置解析、Queue/PriorityClass/Job 引用关系、优先级大小、同节点资源争用、CNI、重启策略、README shell 语法和 `git diff --check`。
