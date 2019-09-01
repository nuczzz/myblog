## kubelet pod驱逐（eviction）源码分析
kubernetes版本：1.13.4

### 什么是kubelet的驱逐？
Kubelet能够监控node上资源消耗，来防止node资源被耗尽。一旦出现资源紧缺的迹象，Kubelet 就会主动终止一或多个 Pod 的运行，以回收紧张的资源。
//todo：监控资源包括哪些？回收资源又包括哪些？

### 驱逐方式
1.软驱逐
kubelet配置参数：--eviction-sof。
其它参数：
--eviction-soft-grace-period，观察期时长，如果观察期内持续能满足驱逐条件，则开始执行kill pod操作。当该参数为0时表示硬驱逐。
--eviction-max-pod-grace-period，kill pod优雅退出超时时间。

2.硬驱逐
kubelet配置参数：--eviction-hard。
只要达到驱逐特性类型定义的指标就直接kill pod（或者image？）

//todo：默认方式？

### 驱逐指标
两种驱逐方式支持以下驱逐指标，相关定义在kubernetes/pkg/kubelet/eviction/api/types.go

指标类型|用途
-|-
memory.available|可用的内存
nodefs.available|系统可用的volume空间
nodefs.inodesFree|inode可用的volume空间
imagefs.available|系统可用的镜像空间
imagefs.inodesFree|inode可用的镜像空间
allocatableMemory.available|可分配给pod的内存大小
pid.available|pid资源

### 避免波动
内部：--eviction-minimum-reclaim。在执行驱逐的时候，为了避免资源在thresholds指标的上下波动，kubelet引入了该参数。该参数定义了每次驱逐操作必须至少清理出多少资源才能够停止执行。（比如：imagefs.available=2Gi）
外部：--eviction-pressure-transition-period。该参数定义了 kubelet 多久才上报api-server当前节点的状态，这将避免scheduler将驱逐的pod立即重新调度到该节点，再次触发资源压力的死循环。

### eviction与QoS（Quality of service）

### 代码目录结构

### 整体逻辑

### 流程细节

### go编程技巧

### linux epoll+event事件驱动机制


