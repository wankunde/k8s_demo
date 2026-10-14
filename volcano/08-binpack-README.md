# 08：Binpack 节点装箱对照实验

Binpack 的目标是把新 Pod 放到**还能容纳它、且资源申请比例更高**的节点，减少资源碎片。本课先关闭、再开启 binpack，对相同的 6 个 worker 观察节点选择是否反转。它使用 CPU/内存 `requests`，不是 `kubectl top` 的实时利用率，也不是实际运行时长或 Pod 数。

沿用 UAT `jscs-ai-works-beta`、命名空间 `infra-demo-prod`、Volcano `v1.15.0`。所有集群命令均通过 `remote-kubectl.sh uat`；候选节点来自当前 UAT，不是通用集群默认值。

## 1. 拓扑与预期

| 对象 | 资源申请 | 节点约束 | 用途 |
| --- | --- | --- | --- |
| seed Job | 1 个 Pod，2 CPU / 64Mi，运行 900 秒 | 固定 A：`10.159.110.29` | 增大 A 的申请比例，贯穿两轮 |
| spread Job | 6 个 Pod，各 100m / 32Mi，运行 120 秒 | A 或 B：`10.159.114.29` | `binpack.weight: 0` 对照 |
| pack Job | 与 spread 完全相同 | 与 spread 完全相同 | `binpack.weight: 3` 实验 |

三个 Job 使用同一个 `volcano-learn-08-binpack` Queue。Queue 不设置 NodeGroup affinity，两个候选节点都必须没有 `volcano.sh/nodegroup-name` 标签，兼容 07 的 `nodegroup.strict: false`。本课不修改节点标签。

2026-10-14 只读检查时，A、B 均为 Ready、未 cordon、无 taint，allocatable 均为 28 CPU / `64374336Ki` 内存。A 当时已申请 4350m CPU、约 27% 内存；B 只有 450m CPU、约 1% 内存。seed 增大这个差异。**每轮仍须重新检查；共享业务变动可能使该预期失效。**

预期：A 持续比 B 更满且两边都能容纳全部 worker 时，spread 的 6 个 worker 应选择 B，pack 的 6 个 worker 应选择 A。关闭 binpack 不意味着必须平均分成 `3:3`；LeastRequested 同样会受原有业务申请影响。

## 2. 为什么要用两份 scheduler 配置

UAT 原配置已经有 `binpack.weight: 3`，但 nodeorder 未显式设置的 LeastRequested、BalancedResource、ImageLocality、PodTopologySpread 等默认评分也在参与；不能把“多数 Pod 在一个节点”直接归因于 binpack。

两份 08 配置基于 07，保留原 actions、Priority/Gang、predicates、proportion、GPU 及 NodeGroup 的硬约束。仅为本实验简化评分：

- nodeorder 只保留 `leastrequested.weight: 1`，其余 7 项显式为 `0`；predicates 仍检查硬节点亲和、资源和 taint。
- NodeGroup 和 colocationnodeorder 的 `enableNodeOrder: false` 排除额外评分，保留 NodeGroup 的 `enablePredicate: true`。
- binpack 的 CPU、内存权重都为 `1`，`binpack.resources: ""` 不加入 GPU 扩展资源。
- spread 和 binpack 两份文件之间，唯一语义差异是 `binpack.weight: 0 → 3`。

设加入当前 worker 后，CPU/内存的申请比例平均值为 `r`，评分的近似关系是：

```text
r = ((usedCPU + podCPU) / allocatableCPU
   + (usedMemory + podMemory) / allocatableMemory) / 2

spread: LeastRequested ≈ 100 × (1 - r)       -> 越空越优先
pack:   LeastRequested + Binpack
      ≈ 100 × (1 - r) + 300 × r
      = 100 + 200 × r                        -> 越满越优先
```

LeastRequested 的实际分值有整数截断，上式用于解释方向，不代替调度日志。资源不足的节点由预选阶段排除，不会因 binpack 分高而强行绑定。已有 Running Pod 不会因为修改评分被搬迁，所以必须使用两个不同名称的新 Job 对照。

**两份文件都是完整的共享 scheduler ConfigMap，会影响整个 UAT 的 Volcano 作业。仅在允许变更评分的实验窗口应用，完成后恢复实时备份。** 不能把完整文件直接覆盖到不同集群，也不能因试验 Pod 仅 `sleep` 就忽略其资源预留和全局评分变化。

## 3. 预检查与备份

以下变量和命令在同一个 Bash 会话中执行。新对象的创建由操作者完成；本文提供命令供执行，本次未向 UAT 写入。

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh
D=/Users/wankun/ws/kunwan/k8s_demo/volcano

"$K" uat get namespace infra-demo-prod -o name
"$K" uat get node 10.159.110.29 10.159.114.29 \
  -L kubernetes.io/hostname,volcano.sh/nodegroup-name \
  -o 'custom-columns=NAME:.metadata.name,HOSTNAME:.metadata.labels.kubernetes\.io/hostname,NODEGROUP:.metadata.labels.volcano\.sh/nodegroup-name,READY:.status.conditions[?(@.type=="Ready")].status,UNSCHEDULABLE:.spec.unschedulable,TAINTS:.spec.taints'
"$K" uat describe node 10.159.110.29 10.159.114.29

# 保存实时正文；使用独立临时目录，避免覆盖前一次实验的备份。
BACKUP_DIR=$(mktemp -d /tmp/volcano-binpack.XXXXXX)
"$K" uat get configmap volcano-scheduler-configmap -n volcano-system \
  -o jsonpath='{.data.volcano-scheduler\.conf}' \
  > "$BACKUP_DIR/before.conf"
test -s "$BACKUP_DIR/before.conf"

diff -u "$BACKUP_DIR/before.conf" \
  <(ruby -ryaml -e 'print YAML.load_file(ARGV[0])["data"]["volcano-scheduler.conf"]' \
    "/Users/wankun/ws/kunwan/k8s_demo/volcano/08-spread-scheduler-config.yaml")
```

检查两节点状态、标签及 `Allocated resources` 的 requests：A 必须能再容纳 seed 和全部 worker，即额外 2600m CPU / 256Mi；B 必须能容纳 600m CPU / 192Mi。若当前拓扑或资源不满足，先调整本地三个 Job 的候选节点并重新验证。不要用 `limits` 或 `kubectl top` 替代 requests 检查。

首次运行应确认四个新名称未被其他实验占用；Job 模板通常不可更新，重跑需要新名称并保留两轮等量请求，不能重新 apply 已完成 Job 期待它重启。

## 4. 对照轮：binpack 关闭

```bash
"$K" uat apply -f - < "/Users/wankun/ws/kunwan/k8s_demo/volcano/08-spread-scheduler-config.yaml"

# 配置投影可能需要约 1–2 分钟；必须核对挂载内容和成功热加载。
"$K" uat exec -n volcano-system deploy/volcano-scheduler -- \
  cat /volcano.scheduler/volcano-scheduler.conf
"$K" uat logs -n volcano-system deploy/volcano-scheduler --since=2m \
  | rg 'Finished loading scheduler config|binpack weight is zero|Failed to get plugin|invalid|Invalid'

# 确认挂载配置为 weight=0，且热加载成功、出现 skip node order function 后继续。
"$K" uat apply -f - < "/Users/wankun/ws/kunwan/k8s_demo/volcano/08-binpack-queue.yaml"
"$K" uat get queue volcano-learn-08-binpack \
  -o 'custom-columns=NAME:.metadata.name,STATE:.status.state,WEIGHT:.spec.weight'
# Queue 必须为 Open。
"$K" uat apply -f - < "/Users/wankun/ws/kunwan/k8s_demo/volcano/08-binpack-seed-job.yaml"
"$K" uat wait --for=condition=Ready pod -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=seed --timeout=120s
"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=seed -o wide

# seed 必须 Running 在 A；保存加入 seed 后、探针提交前的资源快照。
"$K" uat describe node 10.159.110.29 10.159.114.29 \
  > "$BACKUP_DIR/before-spread.nodes.txt"
"$K" uat apply -f - < "/Users/wankun/ws/kunwan/k8s_demo/volcano/08-binpack-spread-job.yaml"
"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=spread -o wide -w
```

若 wait 时 controller 尚未创建 Pod，先通过 get 确认出现 seed Pod，再运行 wait。观察到 6 个 spread worker 均已绑定后结束 watch，保存本轮证据：

```bash
"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=spread -o json \
  > "$BACKUP_DIR/spread.pods.json"
"$K" uat logs -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=spread --prefix=true --tail=10

# 下一轮前，必须看到 Completed / succeeded=6，排除上一轮 requests 的干扰。
"$K" uat get jobs.batch.volcano.sh volcano-learn-08-spread -n infra-demo-prod \
  -o 'custom-columns=NAME:.metadata.name,PHASE:.status.state.phase,SUCCEEDED:.status.succeeded'
```

## 5. 实验轮：binpack 开启

spread 完成后立即进行本轮，确保 seed 仍在 900 秒观察期内。Pod 的 `sleep` 不会阻止调度，只用于保留观察窗口。

```bash
"$K" uat apply -f - < "/Users/wankun/ws/kunwan/k8s_demo/volcano/08-binpack-scheduler-config.yaml"
"$K" uat exec -n volcano-system deploy/volcano-scheduler -- \
  cat /volcano.scheduler/volcano-scheduler.conf
"$K" uat logs -n volcano-system deploy/volcano-scheduler --since=2m \
  | rg 'Finished loading scheduler config|Failed to get plugin|invalid|Invalid'

# 挂载正文必须为 weight=3，热加载成功后再继续。
"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=seed -o wide
"$K" uat describe node 10.159.110.29 10.159.114.29 \
  > "$BACKUP_DIR/before-pack.nodes.txt"

# seed 仍 Running、A 比 B 更满、两边都可容纳全部 worker 才提交。
"$K" uat apply -f - < "/Users/wankun/ws/kunwan/k8s_demo/volcano/08-binpack-pack-job.yaml"
"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=pack -o wide -w
```

结束 watch 后保存结果并汇总两轮的绑定节点数量：

```bash
"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=pack -o json \
  > "$BACKUP_DIR/pack.pods.json"
"$K" uat logs -n infra-demo-prod \
  -l app=volcano-learn,lesson=08,case=pack --prefix=true --tail=10

ruby -rjson -e '
  ARGV.each do |path|
    pods = JSON.parse(File.read(path)).fetch("items")
    raise "expected 6 bound pods: #{path}" unless pods.size == 6 &&
      pods.all? { |p| !p.dig("spec", "nodeName").to_s.empty? }
    counts = pods.group_by { |p| p.dig("spec", "nodeName") }.transform_values(&:size)
    puts "#{File.basename(path)}: #{counts}"
  end
' "$BACKUP_DIR/spread.pods.json" "$BACKUP_DIR/pack.pods.json"
```

## 6. 验收与排错

以下条件全部满足，才能判定本课行为通过：

1. 两份实际生效配置只差 `binpack.weight: 0/3`；nodeorder 仅启用 LeastRequested，额外节点评分关闭。
2. seed 两轮都在 A Running；spread 已 Completed 才提交 pack，两个资源快照均证明 A 的平均申请比例高于 B，且两边都有容量。
3. spread、pack 的镜像、候选节点、Gang 参数和单 Pod requests 相同，各有 6 个 worker；pack 没有单节点 selector 或 preferred affinity。
4. 在本文拓扑和资源差异持续成立时，spread 为 B 上 6 个，pack 为 A 上 6 个；Pod `spec.nodeName` 和日志中的 Downward API 节点一致。
5. 两轮 Job 最终均 `Completed`、6 个 worker 成功退出；资源不足导致的 Pending 或镜像/CNI 错误不能算通过。

若差异反转或只有一个节点能容纳 Pod，本轮结果无法证明 binpack，需重新建立前置条件。若 pack 出现多个节点，先查 requests、业务变动、实际热加载配置及候选节点事件，不要先认定插件失效。

```bash
"$K" uat get jobs.batch.volcano.sh -n infra-demo-prod \
  -l app=volcano-learn,lesson=08 \
  -o 'custom-columns=NAME:.metadata.name,QUEUE:.spec.queue,PHASE:.status.state.phase,SUCCEEDED:.status.succeeded'
"$K" uat get pods -n infra-demo-prod -l app=volcano-learn,lesson=08 -o wide
# 对异常 Pod 使用上一步的精确名称。
# "$K" uat describe pod <精确Pod名称> -n infra-demo-prod
```

源码的 `Binpack score for Task ... on node ...` 为 V(4) 日志，当前 UAT scheduler 为 `-v=3`，因此本课不要求现有日志出现逐 Pod 分值，也不修改 Deployment 的日志级别。如果实验环境已开启 V(4)，可以额外采集这些分值；配置正文、A/B 对照和节点证据仍须保留。

## 7. 恢复

确认 spread、pack 均已完成且没有 Pending 探针，再恢复实验前保存的实时配置。seed 只固定 A，不依赖装箱评分，可以继续自行结束。不要把 07 文件直接作为恢复基线，以免覆盖实验前的其他变更。

```bash
test -s "$BACKUP_DIR/before.conf"
ruby -ryaml -e '
  print({
    "apiVersion" => "v1", "kind" => "ConfigMap",
    "metadata" => {
      "name" => "volcano-scheduler-configmap", "namespace" => "volcano-system"
    },
    "data" => {"volcano-scheduler.conf" => File.read(ARGV[0])}
  }.to_yaml)
' "$BACKUP_DIR/before.conf" | "$K" uat apply -f -

"$K" uat exec -n volcano-system deploy/volcano-scheduler -- \
  cat /volcano.scheduler/volcano-scheduler.conf
"$K" uat logs -n volcano-system deploy/volcano-scheduler --since=2m \
  | rg 'Finished loading scheduler config|invalid|Invalid'
```

如实验失败且有 Pending 探针，先由操作者按精确对象处理，避免恢复后这些 Job 按原评分意外启动。三个 Job 预设 `ttlSecondsAfterFinished: 300`，只有完成后才触发；Queue 不随 Job 清理。本仓库使用的远程技能禁止执行删除，本课不提供删除命令。保留备份与两轮 JSON 作为证据。

## 验证记录

2026-10-14 已通过包装脚本只读确认：UAT 当前配置与 07 基线一致、已有 `binpack.weight: 3`；scheduler 镜像标签为 `v1.15.0`，日志级别 `-v=3`；上述两个候选节点的状态、标签、allocatable 和 requests 符合初始拓扑。

同时核对官方 `v1.15.0` 源码：binpack 使用 `(used + requested) / allocatable`，CPU/内存权重默认都为 `1`，总分乘以 `100 × binpack.weight`；总权重为 `0` 时不注册节点评分函数。nodeorder 的 LeastRequested 使用相同的 CPU/内存等权资源，其他默认评分必须显式关闭。

本地验证：六个 YAML 对象及两份内嵌 scheduler 配置可解析；Job -> Queue 引用、候选节点、CNI、Gang、TTL、两轮等量 requests 与配置差异检查通过；本文 Bash 代码块通过 `bash -n`。公式按初始资源快照计算也符合 spread 选择 B、pack 选择 A 的方向。以上为静态验证，不是调度实跑。

**尚未实跑：**精确预检查显示 `volcano-learn-08-binpack` Queue 和 `infra-demo-prod` 下 `volcano-learn-08-seed`、`volcano-learn-08-spread`、`volcano-learn-08-pack` 三个 Job 均为 `NotFound`。`remote-k8s-operations` 技能要求新对象不存在时停止写入，本次未创建对象、未修改共享 scheduler，未执行服务端 dry-run。真实节点反转和 Job 完成结果需按本文顺序采集。

参考：

- [Volcano v1.15.0 binpack 源码](https://github.com/volcano-sh/volcano/blob/v1.15.0/pkg/scheduler/plugins/binpack/binpack.go)
- [Volcano v1.15.0 nodeorder 源码](https://github.com/volcano-sh/volcano/blob/v1.15.0/pkg/scheduler/plugins/nodeorder/nodeorder.go)
- [远程操作技能及创建限制](/Users/wankun/.codex/skills/remote-k8s-operations/SKILL.md)
