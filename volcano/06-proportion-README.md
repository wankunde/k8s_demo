# 06：Proportion 按 Queue 权重计算资源份额

本课验证 Volcano 的 **`proportion` Queue 插件**：当多个 Queue 同时有超过集群容量的需求时，调度器按 Queue 的 `spec.weight` 计算各自的 `deserved`（应得资源）。本例让 team-a、team-b 的权重为 `1:3`，并给两边制造完全相同的超额需求；预期 scheduler 指标中的 CPU、内存 `deserved` 也接近 `1:3`。

这里不是 `resource-strategy-fit` 插件的 `proportional` 参数。后者用于在节点上为 GPU 保留成比例的 CPU/内存，是另一种功能。

## 安全边界

本例以 05 完成后的 UAT Volcano `v1.15.0` 配置为基线，但只能在已经隔离调度影响的实验环境运行，**不要未经评估直接提交到共享 UAT 或生产集群**：

- 两个需求 Pod 各请求 `1000000` CPU 和 `1Pi` 内存，预期一直 Pending，不会真正启动或消耗计算资源。
- 但这两个 Queue 会参与全局 `deserved` 计算，可能改变其他 Queue 的调度份额；“Pod 没运行”不代表对共享调度器没有影响。
- Pending Job 不会触发 `ttlSecondsAfterFinished`；实验结束后必须由有清理权限的操作者清理本课对象。本仓库使用的远程操作技能禁止执行删除，因此这里不提供或执行自动删除步骤。
- `06-proportion-scheduler-config.yaml` 是完整 ConfigMap，不是局部 patch。它包含环境专用插件，应用前必须与实时配置逐行比对，整段覆盖会影响所有 Volcano 工作负载。

## 1. 前置条件

先读取目标集群的真实 scheduler 配置，确认某个 tier 中存在以下插件；不要仅凭 Queue 有 `weight` 字段就继续：

```yaml
- name: proportion
```

调度器还要开启 metrics。常见的官方配置会在第二个 tier 放置 `predicates`、`proportion`、`nodeorder` 等插件，但应在现有完整配置上做最小合并，不能用一段示例覆盖 ConfigMap。

当前 `04 Queue` 的 UAT 检查记录显示，当时的 `volcano-system/volcano-scheduler-configmap` **没有启用** `proportion`。`06-proportion-scheduler-config.yaml` 继承 `05-preempt-scheduler-config.yaml` 的 `allocate, preempt`、Priority/Gang 设置和所有 UAT 专用插件，只增加一项 `- name: proportion`。

先证明 05 配置已经是当前基线，并核对 05 -> 06 的语义差异：

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

diff -u \
  <("$K" uat get configmap volcano-scheduler-configmap -n volcano-system \
    -o jsonpath='{.data.volcano-scheduler\.conf}') \
  <(ruby -r yaml -e 'print YAML.load_file(ARGV[0])["data"]["volcano-scheduler.conf"]' \
    /Users/wankun/ws/kunwan/k8s_demo/volcano/06-proportion-scheduler-config.yaml)
```

确认后由有共享调度器变更权限的操作者应用，并同时核对挂载文件和热加载日志：

```bash
"$K" uat apply -f - \
  < /Users/wankun/ws/kunwan/k8s_demo/volcano/06-proportion-scheduler-config.yaml

"$K" uat exec -n volcano-system deploy/volcano-scheduler -- \
  cat /volcano.scheduler/volcano-scheduler.conf

"$K" uat logs -n volcano-system deploy/volcano-scheduler --since=5m \
  | rg 'Finished loading scheduler config|actions=|proportion|invalid|Invalid|failed|Failed'
```

ConfigMap 更新成功不等于配置已生效。必须看到 scheduler 成功热加载，并且实际挂载文件中存在 `name: proportion`；出现 invalid/failed 时，v1.15 可能继续使用上一次有效配置。

## 2. 为什么使用“超额但不运行”的需求

`proportion` 的简化计算公式是：

```text
queue.deserved = max(
  queue.guarantee（如果设置）,
  min(
    queue.request,
    queue.capability（如果设置）,
    remaining cluster resource × queue.weight / active queues total weight
  )
)
```

如果两边只请求很少的资源，`deserved` 会先被各自的 `request` 截断，两边可能显示相同值，无法证明权重生效。本例让两边请求都远超集群容量，并且不设置 `capability`，排除这两个干扰因素。

如果只有本例两个活跃 Queue，设集群可分配 CPU 为 `C`：

```text
team-a.deserved ≈ C × 1 / (1 + 3)
team-b.deserved ≈ C × 3 / (1 + 3)
team-b / team-a ≈ 3
```

即使存在其他活跃 Queue，只要 team-a、team-b 都没有被 `request`、`capability` 截断，也没有被 `guarantee` 改写，两者的相对比例仍应接近 `1:3`；其他 Queue 只会改变共同的分母。

## 3. 提交顺序

以下命令沿用本系列的 UAT 别名，只能由已确认调度影响范围和清理方案的操作者执行：

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

"$K" uat apply -f - < /Users/wankun/ws/kunwan/k8s_demo/volcano/06-proportion-queues.yaml
"$K" uat get queues.scheduling.volcano.sh \
  volcano-learn-06-team-a volcano-learn-06-team-b \
  -o 'custom-columns=NAME:.metadata.name,STATE:.status.state,WEIGHT:.spec.weight'

# 两个 Queue 都是 Open 后，再提交相同需求的 Job。
"$K" uat apply -f - < /Users/wankun/ws/kunwan/k8s_demo/volcano/06-proportion-demand.yaml
```

同名 Job 已存在时不要重新 apply 来改变 Pod 模板；VolcanoJob 通常只允许更新 `minAvailable` 和 `tasks[*].replicas`。需要修改模板时，应在隔离环境清理旧对象，或改成新的实验名称。

## 4. 验收证据

先证明 Queue、Job、PodGroup 的关联正确：

```bash
"$K" uat get queues.scheduling.volcano.sh \
  volcano-learn-06-team-a volcano-learn-06-team-b \
  -o 'custom-columns=NAME:.metadata.name,STATE:.status.state,WEIGHT:.spec.weight,ALLOCATED_CPU:.status.allocated.cpu'

"$K" uat get jobs.batch.volcano.sh -n infra-demo-prod \
  -l app=volcano-learn,lesson=06 \
  -o 'custom-columns=NAME:.metadata.name,QUEUE:.spec.queue,PHASE:.status.state.phase'

"$K" uat get podgroups.scheduling.volcano.sh -n infra-demo-prod \
  -o 'custom-columns=NAME:.metadata.name,QUEUE:.spec.queue,PHASE:.status.phase'

"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=06 -o wide
```

预期只有两个 Pending Pod，并出现 `Insufficient cpu` / `Insufficient memory` 一类事件。这是本实验故意制造的状态，不是镜像、CNI 或 Volcano 故障。若 Pod 意外 Running，应立即停止实验并检查请求是否被修改。

最终证据来自 scheduler metrics，而不是 Pod 个数。UAT 的 scheduler Deployment 已开启 `--enable-metrics=true`，但 `volcano-system` 没有 scheduler Service，不能使用 Service proxy。先解析当前 Running scheduler Pod，再通过 Pod proxy 访问 `:8080/metrics`：

```bash
SCHEDULER_POD=$("$K" uat get pods -n volcano-system \
  -l app=volcano-scheduler \
  -o 'jsonpath={.items[?(@.status.phase=="Running")].metadata.name}')

"$K" uat get --raw \
  "/api/v1/namespaces/volcano-system/pods/http:${SCHEDULER_POD}:8080/proxy/metrics" \
  | rg 'volcano_queue_(weight|deserved_milli_cpu|deserved_memory_bytes).*volcano-learn-06-team-[ab]'
```

该 Deployment 当前只有一个副本；如果以后扩容到多个 Running Pod，上面的 JSONPath 会返回多个名称，此时应先选定一个精确 Pod，不能把多个名称拼进 proxy URL。

验收时分别记录：

1. `volcano_queue_weight`：team-a 为 `1`，team-b 为 `3`。
2. `volcano_queue_deserved_milli_cpu`：`team-b / team-a` 约等于 `3`。
3. `volcano_queue_deserved_memory_bytes`：`team-b / team-a` 约等于 `3`。
4. 两个 Job 的请求和 Pod 数相同，且 Queue 没有设置 `capability`；否则不能把比例归因于 `weight + proportion`。

如果指标不存在，先回到前置条件检查 plugin、metrics 和实际 scheduler 版本，不能用 Queue YAML 中的 `weight: 1/3` 代替运行证据。

## 5. 本次完成范围

- `06-proportion-queues.yaml`：两个权重为 `1:3` 的 Queue。
- `06-proportion-demand.yaml`：两个相同的超额需求 Job，用于稳定暴露 `deserved` 指标。
- `06-proportion-scheduler-config.yaml`：继承 05 的完整 UAT 配置，只新增 `proportion` 插件。
- 本文档：配置前置条件、风险边界、提交顺序和可判定的验收标准。

本次只完成本地 YAML 与文档，不声称共享 UAT 已启用插件或已完成真实比例验证。远端实跑必须先确认 05 已生效、调度影响已经隔离，并由操作者接受共享 scheduler 配置变化。

## 验证记录

本次已完成四个 06 文件的 YAML 解析，并检查了 Queue 权重 `1:3`、两个 Job 的等量请求、Job -> Queue 引用、`cni: macvlan` 注解，以及 scheduler 内嵌配置。把 06 的 `proportion` 条目移除后，解析后的 scheduler 配置与 05 完全一致，证明 06 没有覆盖 05 的其他调度语义；README 中的 shell 片段也通过 `bash -n`。

同时核对了 Volcano `v1.15.0` 源码：`proportion` 会按 `weight` 计算 `deserved`，并导出本文使用的 `volcano_queue_weight`、`volcano_queue_deserved_milli_cpu` 和 `volcano_queue_deserved_memory_bytes` 指标。

本次没有修改 UAT scheduler ConfigMap，也没有创建 Queue/Job。共享 scheduler 变更会影响集群其他工作负载；并且 `remote-k8s-operations` 技能要求新对象存在性预检查返回 NotFound 后停止，不能用 server-side dry-run 代替实跑。因而真实热加载、Pending Pod 和 `deserved` 的 `1:3` 指标仍是独立实验环境中的待验证项。

参考：

- Volcano Queue design: <https://github.com/volcano-sh/volcano/blob/master/docs/design/queue/queue.md>
- Volcano v1.15.0 proportion plugin source: <https://github.com/volcano-sh/volcano/blob/v1.15.0/pkg/scheduler/plugins/proportion/proportion.go>
- Volcano v1.15.0 Queue metrics source: <https://github.com/volcano-sh/volcano/blob/v1.15.0/pkg/scheduler/metrics/queue.go>
