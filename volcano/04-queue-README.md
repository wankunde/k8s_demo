# 04：Queue、Job 和 PodGroup

Queue 可以把多个作业归到一个调度组，作为跨作业资源分配、限制和回收策略的作用对象。Queue 是集群级资源，Job 和 PodGroup 是命名空间级资源。Namespace 不会自动决定 Job 属于哪个 Queue；本例两个 Job 都在 `infra-demo-prod`，通过各自的 `spec.queue` 加入不同的 Queue。

关系：`Job.spec.queue -> Queue`；Job 控制器创建 PodGroup，并在 `PodGroup.spec.queue` 中记录对应队列；PodGroup 把属于这个 Job 的 Pod 组织成一个 Gang。

## 当前 UAT 可以验证什么

2026-10-12 只读检查 `volcano-system/volcano-scheduler-configmap`：actions 为 `allocate`，plugins 包含 `priority`、`gang`、`predicates`、`colocationfilter`、`colocationnodeorder`、`gpushare`、`nodeorder`、`binpack`；没有启用 `proportion`，也没有 `reclaim`、`preempt` actions。

本例验证目标是两个 Job 分别归属两个 Queue、PodGroup 的队列字段、Pod 运行及队列资源统计。两个 Queue 配置 `weight: 1` 和 `weight: 3` 是为了学习字段含义；当前配置下不能据此验证 1:3 的资源分配或认定 B 应先运行。即使启用相应插件，没有资源争用时也不应期待看到明显的权重效果。没有为此实验修改共享调度器配置。

| 字段 | 含义 | 本次观察边界 |
| --- | --- | --- |
| `Job.spec.queue` | Job 所属队列的名称 | 核对 Job、PodGroup 的队列字段 |
| `Queue.spec.parent` | 父队列 | 本例两个队列同属 `root` |
| `Queue.spec.weight` | 权重，配合 proportion 等相应策略使用 | 读取配置值，不验证分配比例 |
| `Queue.spec.capability` | 配合相应策略限制队列资源总量 | YAML 中注释保留，未启用、未验证 |
| `Queue.spec.reclaimable` | 是否允许其他队列回收它借用的资源 | 当前没有 reclaim action，不验证回收 |
| `Queue.status.state` | 控制器维护的队列状态 | 创建后检查是否为 Open；当前 UAT 的 `spec` 没有 `state` 字段 |
| `Queue.status.allocated` | 调度器统计的已分配资源 | 观察统计变化；不等于容器实时 CPU/内存用量 |

## 1. 提交 Queue，再提交 Job

以下命令由你手工执行。两个 Job 总共只请求 `200m` CPU、`64Mi` 内存，各运行约 120 秒。没有节点选择条件，不依赖上一个实验的 `volcano-observe-02` 标签。

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

"$K" uat apply -f - < /Users/wankun/ws/kunwan/k8s_demo/volcano/04-queues.yaml

# 先确认两个队列都已创建，状态为 Open，再继续提交 Job。
"$K" uat get queues.scheduling.volcano.sh -l app=volcano-learn,lesson=04

"$K" uat apply -f - < /Users/wankun/ws/kunwan/k8s_demo/volcano/04-queue-jobs.yaml
```

同名 Job 尚存在时，重新 apply 不会重新执行已完成的任务。下一轮实验可把两个 Job 的 `metadata.name` 改成新名称，继续引用相同的 Queue。

## 2. 在 k9s 中观察

使用 UAT context，先用 `:ns` 选中 `infra-demo-prod`。

1. 输入 `:queues.scheduling.volcano.sh`，再输入 `/volcano-learn-04`。选中一个 Queue，按 `y` 看 `spec.weight`、`status.state`、`status.allocated`，按 `d` 查看详情。
2. 输入 `:vcjob`，输入 `/volcano-learn-04`，按 `y` 核对两个 Job 的 `spec.queue`。
3. 输入 `:podgroups.scheduling.volcano.sh`，输入 `/volcano-learn-04`，按 `y` 核对 `spec.queue` 和 `status.phase`。
4. 输入 `:pods`，输入 `/volcano-learn-04`。运行期间按 `l` 看日志，按 `d` 查看调度事件。

有可用资源时，两边都可以运行；Queue 不会自动把作业限制成一次运行一个。

## 3. 用命令核对证据

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

"$K" uat get queues.scheduling.volcano.sh -l app=volcano-learn,lesson=04 \
  -o 'custom-columns=NAME:.metadata.name,STATE:.status.state,WEIGHT:.spec.weight,CPU:.status.allocated.cpu,MEMORY:.status.allocated.memory'

"$K" uat get jobs.batch.volcano.sh -n infra-demo-prod -l app=volcano-learn,lesson=04 \
  -o 'custom-columns=NAME:.metadata.name,QUEUE:.spec.queue,PHASE:.status.state.phase'

# PodGroup 名称/标签由控制器生成；用完整列表中的名字和队列确认关联。
"$K" uat get podgroups.scheduling.volcano.sh -n infra-demo-prod \
  -o 'custom-columns=NAME:.metadata.name,QUEUE:.spec.queue,MINMEMBER:.spec.minMember,PHASE:.status.phase'

"$K" uat get pods -n infra-demo-prod -l app=volcano-learn,lesson=04 -o wide

# 任一 Job 卡住时，先看对象详情和 Pod 事件，不把 Pending 自动归因于队列权重。
"$K" uat describe jobs.batch.volcano.sh volcano-learn-04-team-a-job -n infra-demo-prod
"$K" uat describe pods -n infra-demo-prod -l app=volcano-learn,lesson=04
"$K" uat logs -n infra-demo-prod -l app=volcano-learn,lesson=04 --prefix=true --tail=20
```

预期观察顺序是 Pod 等待调度、运行约两分钟、Job 结束。具体阶段可能很快跳过，不能要求每个阶段都能在 k9s 捕捉到。队列统计也可能在 Pod 运行前出现变化，需要与 Pod 的节点分配和实际状态一起判断。

## 4. 结束与进阶

Job 设置 `ttlSecondsAfterFinished: 300`，结束后由控制器清理；Queue 会保留。当前远程操作技能禁止执行删除，所以本例没有自动删除 Queue 的脚本。

下一步在独立实验环境中启用 `proportion` 并构造资源争用，再比较权重和 `capability`；之后再学习 `reclaim` 和 `preempt`。这些需要调度器配置和竞争负载，本例暂不启用。

## 验证记录

本次检查了 UAT 的实际调度配置和 Queue schema，并完成本地 YAML 解析与字段一致性检查。尚未在 UAT 创建这四个新对象，未验证真实 Pod 运行、Queue 统计变化或 TTL 清理。

四个精确名称的存在性预检查均返回 NotFound。[remote-k8s-operations 技能](/Users/wankun/.codex/skills/remote-k8s-operations/SKILL.md) 对新对象要求："report that and do not run the operation"，因此本次没有执行创建，也没有用 server-side dry-run 代替存在性检查。上面的提交命令供你手工执行。
