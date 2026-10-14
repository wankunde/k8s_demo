# 07：用 NodeGroup 把 Queue 绑定到机器资源池

本课验证 Volcano `nodegroup` 插件的硬隔离和兼容行为。三个 VolcanoJob 只设置 `spec.queue`，Pod 模板不设置 `nodeSelector` 或 Pod `nodeAffinity`：两个绑定 NodeGroup 的 Queue 只能进入对应资源池；第三个不设置 NodeGroup affinity 的 Queue 只能进入未标记节点。

绑定资源池的链路是：

```text
Job.spec.queue
  -> Queue.spec.affinity.nodeGroupAffinity.required...
  -> Node.metadata.labels[volcano.sh/nodegroup-name]
  -> Pod.spec.nodeName
```

未绑定 Queue 的链路是：

```text
Job.spec.queue
  -> Queue 没有 spec.affinity
  -> nodegroup.arguments.strict=false
  -> 只能使用没有 volcano.sh/nodegroup-name 标签的节点
```

## UAT 实验拓扑

2026-10-14 初始只读检查显示，UAT 有 11 个 Ready 节点，现有机器池标签为 `pool=coeus`（10 个）和 `pool=paas`（1 个），当时均没有 `volcano.sh/nodegroup-name` 标签。不能直接给全部节点增加 NodeGroup 标签：启用插件后，没有 affinity 的普通 Queue 不能使用带 NodeGroup 标签的节点。

本课仅从两个真实机器池各取一个节点作为隔离实验切片，其余 9 个节点保持未标记，继续服务普通 Queue：

| 真实机器池 | 实验节点 | NodeGroup | 绑定 Queue |
| --- | --- | --- | --- |
| `coeus` | `10.159.192.50` | `uat-demo-coeus` | `volcano-learn-07-coeus` |
| `paas` | `10.156.193.25` | `uat-demo-paas` | `volcano-learn-07-paas` |
| 未绑定对照组 | 其余 9 个节点之一 | 无标签 | `volcano-learn-07-unbound` |

两个节点在检查时均为 Ready、未 cordon、无 taint。运行前仍须重新核对，因为节点状态和业务负载会变化。

## 1. 前置检查与备份

所有集群命令都通过 UAT 远程包装脚本执行：

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

# 精确确认两个目标节点仍存在。
"$K" uat get node 10.159.192.50 -o name
"$K" uat get node 10.156.193.25 -o name

# 两个节点必须 Ready、未 cordon、无 taint，且真实 pool 标签没有变化。
"$K" uat get node 10.159.192.50 10.156.193.25 \
  -L pool,volcano.sh/nodegroup-name \
  -o 'custom-columns=NAME:.metadata.name,POOL:.metadata.labels.pool,NODEGROUP:.metadata.labels.volcano\.sh/nodegroup-name,READY:.status.conditions[?(@.type=="Ready")].status,UNSCHEDULABLE:.spec.unschedulable,TAINTS:.spec.taints'

# 允许下面两个节点已经具有本文约定的值；发现其他 NodeGroup 或不同值时，
# 先确认所有者和用途，不要覆盖。
"$K" uat get nodes -L volcano.sh/nodegroup-name

# 只保存配置正文，用于实验后恢复；不保存会过期的 resourceVersion/managedFields。
"$K" uat get configmap volcano-scheduler-configmap -n volcano-system \
  -o jsonpath='{.data.volcano-scheduler\.conf}' \
  > /tmp/volcano-scheduler.before-nodegroup.conf
```

## 2. 启用 nodegroup 插件

`07-nodegroup-scheduler-config.yaml` 基于 2026-10-14 UAT 实时配置，只新增以下内容：

```yaml
- name: nodegroup
  enablePredicate: true
  enableNodeOrder: true
  arguments:
    strict: false
```

`enablePredicate: true` 提供硬约束。`strict: false` 是共享 UAT 的必要保护：无 affinity Queue 可以继续使用未打 NodeGroup 标签的节点；但它们不能使用已标记的两个实验节点。

先核对差异，再由有共享调度器变更权限的操作者应用：

```bash
diff -u \
  <("$K" uat get configmap volcano-scheduler-configmap -n volcano-system \
    -o jsonpath='{.data.volcano-scheduler\.conf}') \
  <(ruby -r yaml -e 'print YAML.load_file(ARGV[0])["data"]["volcano-scheduler.conf"]' \
    /Users/wankun/ws/kunwan/k8s_demo/volcano/07-nodegroup-scheduler-config.yaml)

"$K" uat apply -f - \
  < /Users/wankun/ws/kunwan/k8s_demo/volcano/07-nodegroup-scheduler-config.yaml
```

ConfigMap 更新不代表插件已经生效。必须核对 scheduler 挂载文件和热加载日志：

```bash
"$K" uat exec -n volcano-system deploy/volcano-scheduler -- \
  cat /volcano.scheduler/volcano-scheduler.conf

"$K" uat logs -n volcano-system deploy/volcano-scheduler --since=5m \
  | rg 'Finished loading scheduler config|nodegroup|Failed to get plugin|invalid|Invalid|failed|Failed'
```

如果出现 `Failed to get plugin nodegroup`，说明当前镜像没有注册插件，必须停止实验并恢复配置；不能继续给节点打标签或提交 Queue。

## 3. 划分两个实验 NodeGroup

只有插件成功加载后才标记节点：

```bash
"$K" uat label node 10.159.192.50 \
  volcano.sh/nodegroup-name=uat-demo-coeus --overwrite

"$K" uat label node 10.156.193.25 \
  volcano.sh/nodegroup-name=uat-demo-paas --overwrite

"$K" uat get node 10.159.192.50 10.156.193.25 \
  -L pool,volcano.sh/nodegroup-name
```

预期映射：

```text
10.159.192.50  pool=coeus  nodegroup=uat-demo-coeus
10.156.193.25  pool=paas   nodegroup=uat-demo-paas
```

## 4. 创建 Queue 和 Job

先创建 Queue，并确认三个 Queue 都是 `Open`。第三个 Queue 的 `REQUIRED_NODEGROUP` 应为空：

```bash
"$K" uat apply -f - \
  < /Users/wankun/ws/kunwan/k8s_demo/volcano/07-nodegroup-queues.yaml

"$K" uat get queue \
  volcano-learn-07-coeus volcano-learn-07-paas volcano-learn-07-unbound \
  -o 'custom-columns=NAME:.metadata.name,STATE:.status.state,REQUIRED_NODEGROUP:.spec.affinity.nodeGroupAffinity.requiredDuringSchedulingIgnoredDuringExecution'
```

再提交三个 Job。每个 Pod 只请求 `10m CPU / 16Mi`，运行约 120 秒：

```bash
"$K" uat apply -f - \
  < /Users/wankun/ws/kunwan/k8s_demo/volcano/07-nodegroup-jobs.yaml

"$K" uat get jobs.batch.volcano.sh -n infra-demo-prod \
  -l app=volcano-learn,lesson=07 \
  -o 'custom-columns=NAME:.metadata.name,QUEUE:.spec.queue,PHASE:.status.state.phase'

"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=07 -o wide -w
```

## 5. 验收标准

先证明 Pod 模板没有自己的节点选择约束：

```bash
"$K" uat get jobs.batch.volcano.sh -n infra-demo-prod \
  volcano-learn-07-coeus-job volcano-learn-07-paas-job \
  volcano-learn-07-unbound-job \
  -o 'custom-columns=NAME:.metadata.name,QUEUE:.spec.queue,NODE_SELECTOR:.spec.tasks[0].template.spec.nodeSelector,NODE_AFFINITY:.spec.tasks[0].template.spec.affinity.nodeAffinity'
```

再核对实际调度结果和容器日志：

```bash
"$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=07 \
  -o 'custom-columns=NAME:.metadata.name,POOL:.metadata.labels.pool,NODE:.spec.nodeName,PHASE:.status.phase'

"$K" uat logs -n infra-demo-prod \
  -l app=volcano-learn,lesson=07 --prefix=true --tail=20

# 动态找到未绑定 Queue 的 Pod 实际节点，并核对该节点没有 NodeGroup 标签。
UNBOUND_NODE=$("$K" uat get pods -n infra-demo-prod \
  -l app=volcano-learn,lesson=07,case=unbound \
  -o jsonpath='{.items[0].spec.nodeName}')

test -n "$UNBOUND_NODE"
"$K" uat get node "$UNBOUND_NODE" -L pool,volcano.sh/nodegroup-name

# 未绑定 Queue 不能占用两个已标记的实验节点。
case "$UNBOUND_NODE" in
  10.159.192.50|10.156.193.25)
    echo "FAIL: unbound queue entered a labeled NodeGroup: $UNBOUND_NODE" >&2
    false
    ;;
  *)
    echo "PASS: unbound queue used unlabeled node: $UNBOUND_NODE"
    ;;
esac
```

通过条件必须全部满足：

1. coeus Job 的 `spec.queue` 是 `volcano-learn-07-coeus`，Pod 落在 `10.159.192.50`。
2. paas Job 的 `spec.queue` 是 `volcano-learn-07-paas`，Pod 落在 `10.156.193.25`。
3. unbound Job 的 `spec.queue` 是 `volcano-learn-07-unbound`，Queue 没有 `spec.affinity`；Pod 落在其余未标记节点之一，实际节点的 NodeGroup 标签为空。
4. 三个 Job 的 Pod 模板都没有 `nodeSelector` 和 Pod `nodeAffinity`。
5. Pod 日志中的用例、节点与上述映射一致。
6. scheduler 日志没有 `Failed to get plugin nodegroup` 或配置解析错误。

只看到 Pod `Running` 不足以证明 Queue 绑定；必须同时保留 Queue affinity、节点 NodeGroup 标签和 Pod 实际节点三层证据。

## 6. 恢复边界

等待三个 Job 都 `Completed` 后再恢复，避免 Pending Job 在插件关闭后失去资源池约束：

```bash
# 从实验前保存的配置正文生成一个干净 ConfigMap，再恢复。
ruby -ryaml -e '
  conf = File.read(ARGV[0])
  print({
    "apiVersion" => "v1",
    "kind" => "ConfigMap",
    "metadata" => {
      "name" => "volcano-scheduler-configmap",
      "namespace" => "volcano-system"
    },
    "data" => {"volcano-scheduler.conf" => conf}
  }.to_yaml)
' /tmp/volcano-scheduler.before-nodegroup.conf | "$K" uat apply -f -

# 确认 scheduler 已热加载不含 nodegroup 的原配置后，再移除实验标签。
"$K" uat label node 10.159.192.50 volcano.sh/nodegroup-name-
"$K" uat label node 10.156.193.25 volcano.sh/nodegroup-name-

"$K" uat get nodes -L volcano.sh/nodegroup-name
```

Job 完成 300 秒后由 TTL 控制器清理；Queue 不会随 Job 自动清理。本仓库使用的远程操作技能禁止执行删除，因此不提供或执行 Queue 删除命令，剩余三个 Queue 由具备清理权限的操作者按精确名称处理。

## 验证记录

本课文件生成前已只读确认：UAT scheduler 为 `v1.15.0`，Queue CRD 支持 `spec.affinity.nodeGroupAffinity`。Volcano v1.15.0 源码确认 `nodegroup` 已注册，标签键为 `volcano.sh/nodegroup-name`；`strict: false` 下，无 affinity Queue 只能使用未标记节点。

本地检查已经通过：三个 YAML 文件共 7 个对象均可解析；07 scheduler 配置移除 `nodegroup` 条目后与 06/UAT 基线完全一致；两个绑定 Queue 的 required NodeGroup 映射正确，未绑定 Queue 没有 `spec.affinity`；三个 Job 均使用 `schedulerName: volcano`、`cni: macvlan`，且 Pod 模板不含 `nodeSelector` 或 `affinity`；本文所有 Bash 代码块通过 `bash -n`。

2026-10-14 本次补充 unbound 用例时重新只读核对，UAT 已处于部分应用状态：实时 scheduler 配置已包含 `nodegroup`、`enablePredicate: true`、`enableNodeOrder: true`、`strict: false`；`10.159.192.50` 和 `10.156.193.25` 已分别带有 `uat-demo-coeus`、`uat-demo-paas` 标签；两个绑定 Queue 已存在、状态为 `Open`，其 required NodeGroup 与本文一致。

精确预检查同时确认：新增的 `volcano-learn-07-unbound` Queue 和三个 07 Job 均为 `NotFound`。本仓库使用的远程操作技能要求批次中任一新对象不存在时停止整个变更批次，所以本次没有执行远端写入。unbound 兼容性测试及三类 Pod 的节点绑定仍须按本文顺序实跑，并以 Pod 的 `spec.nodeName` 和实际节点标签作为最终验收结果。

参考：<https://github.com/volcano-sh/volcano/blob/v1.15.0/docs/user-guide/how_to_use_nodegroup_plugin.md>
