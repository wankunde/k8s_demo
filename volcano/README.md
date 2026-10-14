# Volcano 调度器学习实验

04：继续学习 Queue，见 [Queue demo](04-queue-README.md)。示例使用 UAT、`infra-demo-prod`，包含两个 Queue 和两个 Job，并说明当前调度插件下可以验证的行为。

05：学习优先级抢占，见 [Preempt demo](05-preempt-README.md)。示例包含 scheduler 配置、专用 Queue、两个 PriorityClass，以及同节点资源争用的 victim/preemptor Job。

06：学习 Queue 权重份额，见 [Proportion demo](06-proportion-README.md)。示例在 05 配置上启用 `proportion`，用相同的超额需求和 scheduler metrics 验证 `weight: 1:3` 对应的 `deserved` 比例。

07：学习 Queue 与机器资源池绑定，见 [NodeGroup demo](07-nodegroup-README.md)。示例启用 `nodegroup`，从 UAT 的 coeus、paas 机器池各取一个节点形成隔离切片；验证两个绑定 Queue 只能进入对应 NodeGroup，同时验证未指定 NodeGroup 的 Queue 只能使用未标记节点。

环境：UAT `jscs-ai-works-beta`，命名空间 `infra-demo-prod`。这里的 `uat` 是远程 kubectl 包装脚本中的集群别名。本集群的 CNI 要求 Pod 标注 `cni: macvlan`。以下例子按顺序运行，每个容器会自行结束。

```bash
K=/Users/wankun/.codex/skills/remote-k8s-operations/scripts/remote-kubectl.sh

# 01：普通 Pod 指定 Volcano 调度器；看 spec.schedulerName、分配的节点和事件。
$K uat apply -f - < volcano/01-pod.yaml
$K uat get pod volcano-learn-01-busybox -n infra-demo-prod -o wide
$K uat describe pod volcano-learn-01-busybox -n infra-demo-prod

# 02：两个 worker 的 minAvailable=2，观察 VolcanoJob、PodGroup 和 Pod。
$K uat apply -f - < volcano/02-gang-job.yaml
$K uat get job.batch.volcano.sh volcano-learn-02-gang-busybox -n infra-demo-prod
$K uat get podgroup -n infra-demo-prod
$K uat get pods -n infra-demo-prod -l volcano.sh/job-name=volcano-learn-02-gang-busybox -o wide

# label node volcano-observe-02=ready
$K uat label node 10.159.192.50 volcano-observe-02=ready
# Unlable node
$K uat label node 10.159.192.50 volcano-observe-02-

# 03：复用现有 Queue 与低优先级 PriorityClass，观察 Pod 字段和队列注解。
$K uat apply -f - < volcano/03-priority-job.yaml
$K uat get job.batch.volcano.sh volcano-learn-03-priority-busybox -n infra-demo-prod
$K uat get pods -n infra-demo-prod -l volcano.sh/job-name=volcano-learn-03-priority-busybox -o 'custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority,CLASS:.spec.priorityClassName,QUEUE:.metadata.annotations.volcano\.sh/queue-name,PHASE:.status.phase'
```

01 的关键字段是 Pod 的 `spec.schedulerName: volcano`。02 的 `minAvailable: 2` 表示两个 worker 必须一起满足准入条件，适合分布式训练；PodGroup 是 Volcano 管理这组 Pod 的对象。03 使用已有的 `coeus-job-low` PriorityClass（配置值为 `-50`）和 `default` Queue。Pod 模板需要声明 `priorityClassName` 才能在 Pod 上看到该类；本集群生成的 Pod 没有填充 `spec.priority` 数值，因此这里只验证类名和队列传递，未验证争用时的排序或抢占。
