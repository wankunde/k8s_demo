1. VCJob , PodGroup , Pod 之间的关系
- VCJob (:vcjob) 是 Volcano 的 Job 对象，定义了作业的整体需求和调度策略。
- PodGroup (:pg) 是 Volcano 用于管理一组相关 Pod 的对象，确保它们按照 Gang Scheduling 的要求一起调度。
- Pod 是实际运行的容器实例，属于某个 PodGroup，并受 VCJob 的调度策略约束。
2. Gang Scheduling 的 minAvailable 字段
- VCJob 的 minAvailable 字段定义了整个作业能够启动运行所需的最少 Pod 总数。
- 每个任务（task）也可以定义自己的 minAvailable 字段，表示该特定任务需要达到的最小可用副本数。
- 当 VCJob 的 minAvailable 和任务的 minAvailable 都满足时，Gang Scheduling 才会允许这些 Pod 一起调度运行。
3. 调度器和队列
- VCJob 的 schedulerName 字段指定了使用的调度器，例如 "volcano"。
- VCJob 的 queue 字段指定了作业所属的队列，调度器会根据队列的优先级和资源分配策略来调度作业。
