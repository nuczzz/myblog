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

### 几种pod类型
1.critical pod
2.static pod


### ephemeral storage
Kubernetes在1.8的版本中引入了一种类似于CPU，内存的新的资源模式：ephemeral-storage，并且在1.10的版本在kubelet中默认打开了这个特性。ephemeral-storage是为了管理和调度Kubernetes中运行的应用的短暂存储。在每个Kubernetes的节点上，kubelet的根目录(默认是/var/lib/kubelet)和日志目录(/var/log)保存在节点的主分区上，这个分区同时也会被Pod的EmptyDir类型的volume、容器日志、镜像的层、容器的可写层所占用。ephemeral-storage便是对这块主分区进行管理，通过应用定义的需求(requests)和约束(limits)来调度和管理节点上的应用对主分区的消耗。
对于未配置单独的imagefs节点上的container而言，ephemeral storage包括root和logs

### emptyDir
EmptyDir类型的volume创建于pod被调度到某个宿主机上的时候，而同一个pod内的容器都能读写EmptyDir中的同一个文件。一旦这个pod离开了这个宿主机，EmptyDirr中的数据就会被永久删除。所以目前EmptyDir类型的volume主要用作临时空间，比如Web服务器写日志或者tmp文件需要的临时目录

### local storage
包含emptyDir、phemeral sotrage、root（imageFs可写层）、logs、本地卷


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


