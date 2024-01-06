# k8s: 到底谁才是草台班子？


大家在对 2023 年诸多互联网公司故障的总结中多次提到了控制 "爆炸半径"，几乎都在说缩小集群规模，那除了缩小集群规模外还有没有其他办法呢？如果一出问题就通过缩小规模去解决，多少会显得有点不够专业（草台班子）。k8s 已经经历了九年半的发展，众多的终端用户在以什么样的方式使用 k8s，即便社区高手如云，也很难把所有使用场景都考虑到并且处理好，但也不至于差到连我们这群"草台班子"都能想到的一些最基本的问题（比如控制爆炸半径）都想不到。比起把集群搞大出问题的人，反而是在出问题后只会喊控制集群规模的那些 k8s 相关的云原生专家们，那些 k8s 集群管理员们，更像是草台班子。（并没有说 k8s 等于云原生的意思，但只要做的事情和 k8s 沾点边就号称云原生，这是事实）

k8s 已经提供了一些用来保障自身及运行于其中的服务的 SLO 的能力，本篇围绕相关参数进行介绍。部分参数可能已经从启动参数中挪到了对应组件的 Configuration 资源中或者只存在于 Configuration 资源中，有需要请参考官方文档进行设置，基于 v1.29。

# Common

> `--http2-max-streams-per-connection` 

服务器为客户端提供的 HTTP/2 连接中最大流数的限制，零表示使用 GoLang 的默认值（250）。这个参数涉及到一个安全漏洞和连接数，详情参考[连接数也会影响 kube-apiserver 内存](https://mp.weixin.qq.com/s/2HkxVu-OjtG2_YKwMAhO0A)。并非只有 kube-apiserver 暴露了这个参数，其他组件对外暴露 API 的同样也暴露了这个参数。

kube-apiserver 早期为 `MaxConcurrentStreams` 设置了默认值 250，并且暴露了参数可以在外部修改，而在 v1.29 的发布中，将其默认值修改为了 100，同时 backport 回了从 v1.25 及之后的所有的版本，这个修改和 golang 的安全漏洞 **[CVE-2023-44487](https://github.com/advisories/GHSA-qppj-fm5r-hxr3)** and **[CVE-2023-39325](https://github.com/advisories/GHSA-4374-p667-p6c8)** 有关。**生产环境一般情况下保持默认值即可**。

> `--kube-api-burst`  default 400
>
> `--kube-api-qps` default 200

访问 kube-apiserver 时的客户端限流设置，尤其注意 kube-scheduler，这两个值会影响 binding 操作，进而影响到调度吞吐，**生产环境建议按需设置，尤其是 kube-scheduler**。

# kube-apiserver

## Rate Limit

> `--enable-priority-and-fairness` default true

如果为 true 且启用了 `APIPriorityAndFairness` 特性门控， 则使用增强的处理程序替换 max-in-flight 处理程序， 以便根据优先级和公平性完成排队和调度，**生产环境强烈建议开启并按需做好对应的配置**。

> `--max-mutating-requests-inflight` default 200

如果 --enable-priority-and-fairness 为 true，那么此值和 --max-requests-inflight 的和将确定服务器的总并发限制（必须是正数）。 否则，该值限制同时运行的变更类型的请求的个数上限。0 表示无限制。

> `--max-requests-inflight` default 400

 如果 --enable-priority-and-fairness 为 true，那么此值和 --max-mutating-requests-inflight 的和将确定服务器的总并发限制（必须是正数）。 否则，该值限制进行中非变更类型请求的最大个数，零表示无限制。

## Reliability

> `--etcd-servers-overrides`

etcd 服务器针对每个资源的重载设置，以逗号分隔。 单个替代格式：组/资源#服务器（group/resource#servers）， 其中服务器是 URL，以分号分隔。 注意，此选项仅适用于编译进此服务器二进制文件的资源。

笔者曾经修复了一个相关的 bug 并随着 v1.25 发布，该 bug 会导致即使通过此参数配置了 event 使用单独的 etcd 存储，kube-controller-manager 中的 gc controller 也需要在启动时创建对应的 event informer，这回带来两个问题：一个 kube-controller-manager 内存使用过高，二是在 event etcd 出现问题时，如果 kube-controller-manager 发生了重启，则即使存其他资源的 etcd 正常，kube-controller-manager 也无法正常启动工作，因为 event informer 会同步失败。详情可以参考 [feat: ignore all event resource for gc](https://github.com/kubernetes/kubernetes/pull/110888)。**生产环境建议大规模集群中通过此参数把 event 拆到单独的 etcd 中存储**。

> `--goaway-chance`

为防止 HTTP/2 客户端卡在单个 API 服务器上，随机关闭某连接（GOAWAY）。其原理是随机给 HTTP/2 客户端发送 GOAWAY 帧，客户端收到后如果有新的请求，则会创建新的连接，而不会继续复用之前的连接，客户端的其他运行中请求不会受到影响。新连接被负载均衡后可能会与其他 API 服务器开始通信。 此参数设置将被发送 GOAWAY 指令的请求的比例。 只有一个 API 服务器或不使用负载均衡器的集群不应启用此特性。 最小值为 0（关闭），最大值为 0.02（1/50 请求）。

> `--min-request-timeout` default 1800

表示处理程序在请求超时前，必须保持连接处于打开状态的最小秒数。 当前只对监听（Watch）请求的处理程序有效。如果是通过 informer 发起的 Watch 请求，默认 5m ~ 10m 的超时时间。

> `--request-timeout` default 1m

表示处理程序在超时之前必须保持打开请求的持续时间。 这是请求的默认请求超时，但对于 Watch 请求，会被 `--min-request-timeout`  标志覆盖。**生产环境建议按需设置，尤其在单个资源类型数据量较大时**，如果在 timeout 时间内没有执行完 List 请求，会触发 informer 的 Relist 操作，严重影响 kube-apiserver 内存，甚至 OOM。

> `--watch-cache` default true

在 API 服务器中启用 watch cache。

## Other

> `--anonymous-auth` default true

启用针对 API 服务器的安全端口的匿名请求。 未被其他身份认证方法拒绝的请求被当做匿名请求。 匿名请求的用户名为 system:anonymous， 用户组名为 system:unauthenticated，**生产环境建议关闭**。

> `--default-not-ready-toleration-seconds` default 300
>
> `--default-unreachable-toleration-seconds` default 300

对污点 NotReady:NoExecute 和 Unreachable:NoExecute 的容忍时长（以秒计）。 开启 `DefaultTolerationSeconds` feature-gate 的情况下，上述两个容忍度之一会被按需添加到尚未具有此容忍度的每个 pod 中。在节点出现上述异常时会影响其上的 Pod 的运行情况，具体在下面会解释。

# kube-controller-manager

## Pod Deletion

> `--unhealthy-zone-threshold` default 0.55

仅当给定区域中处于非就绪状态的节点（最少 3 个）的占比高于此值时， 才将该区域视为不健康。

> `--large-cluster-size-threshold` default  50

node-lifecycle-controller 在执行 Pod 驱逐操作逻辑时， 基于此标志所设置的节点个数阈值来判断所在集群是否为大规模集群。 当集群规模小于等于此规模时， `--secondary-node-eviction-rate` 会被隐式重设为 0。

> `--node-eviction-rate` default 0.1

当某区域健康时，在节点故障的情况下每秒删除 Pods 的节点数。请参阅 `--unhealthy-zone-threshold` 以了解“健康”的判定标准。 这里的区域（zone）在集群并不跨多个区域时指的是整个集群。0.1 代表每 10s 驱逐一台节点，在 Taint Based Eviction 模式下，这个频率是用来控制为 Node 设置 Taint 的，并不是控制最终驱逐 Pod 的。

> `--node-monitor-grace-period`  default 40s

在将一个 Node 标记为不健康之前允许其无响应的时长上限。 必须比 kubelet 的 nodeStatusUpdateFrequency 大 N 倍； 这里 N 指的是 kubelet 发送节点状态的重试次数。

> `--node-monitor-period` default 5s

node-lifecycle-controller 对节点状态进行同步的周期。

> `--node-startup-grace-period` default 1m

在节点启动期间，节点可以处于无响应状态； 但超出此标志所设置的时长仍然无响应则该节点被标记为不健康。

> `--secondary-node-eviction-rate` default 0.01

当一个区域不健康造成节点失效时，每秒钟从此标志所给的节点上删除 Pod 的节点个数。 参见 `--unhealthy-zone-threshold` 以了解“健康与否”的判定标准。 在只有一个区域的集群中，区域指的是整个集群。如果集群规模小于 `--large-cluster-size-threshold` 所设置的节点个数时， 此值被隐式地重设为 0。

**以上参数均与 Node 状态异常时如何处理其上运行中的 Pod 有关，需要格外注意，合理设置这些参数可以有效的控制故障域的大小或者说控制所谓的爆炸半径**。上面提到的 zone 可以通过为 Node 添加如下 Label 来设置：

- failure-domain.beta.kubernetes.io/region (deprecated)

  **Note:** Starting in v1.17, this label is deprecated in favor of topology.kubernetes.io/region.

- failure-domain.beta.kubernetes.io/zone (deprecated)

  **Note:** Starting in v1.17, this label is deprecated in favor of topology.kubernetes.io/zone.

> `--terminated-pod-gc-threshold` default 12500

在已终止 Pod 垃圾收集器删除已终止 Pod 之前，可以保留的已终止 Pod 的个数上限。 若此值小于等于 0，则相当于禁止垃圾回收已终止的 Pod。主要影响 Job 创建出来的 Pod，或者不按照官网文档指导非得给非 Job 的 Pod 设置 activeDeadlineSeconds 的 Pod。影响集群中的 Pod 数量，可能会存在大量 Completed 的 Pod，如果客户端使用不当，有可能给 kube-apiserver 内存造成压力。**生产环境建议调小这个值。**

## Performance

> --concurrent-*-syncs

这是一类参数的统称，代表对应资源的 controller 并行执行时的 worker 数量，按需调整即可，影响 workload 的处理速度。

# Kube-scheduler

除了 Common 提到的参数外，这个组件本身没有什么需要特别关注的参数。正常情况下即使他挂了，也不影响存量业务的运行，一个极端情况需要注意，那就是在本身业务或者平台故障恢复时如果调度器不能用的话，会影响故障恢复速度。但这其实也说明业务或者平台自身架构设计就有问题。

# Kubelet

## Eviction

> `--eviction-hard` default imagefs.available<15%,memory.available<100Mi,nodefs.available<10%

触发 Pod 驱逐操作的一组硬性门限（例如：`memory.available<1Gi` （内存可用值小于 1G）设置。在 Linux 节点上，默认值还包括 `nodefs.inodesFree<5%`。

> `--eviction-minimum-reclaim`

当某资源压力过大时，kubelet 将执行 Pod 驱逐操作。 此参数设置软性驱逐操作需要回收的资源的最小数量（例如：`imagefs.available=2Gi`）。

> `--eviction-pressure-transition-period` default 5m

kubelet 在驱逐压力状况解除之前的最长等待时间。

> `--eviction-max-pod-grace-period`

响应满足软性驱逐阈值（Soft Eviction Threshold）而终止 Pod 时使用的最长宽限期（以秒为单位）。 如果设置为负数，则遵循 Pod 的指定值。 

> `--eviction-soft`

设置一组驱逐阈值（例如：`memory.available<1.5Gi`）。 如果在相应的宽限期内达到该阈值，则会触发 Pod 驱逐操作。

> `--eviction-soft-grace-period`

设置一组驱逐宽限期（例如，`memory.available=1m30s`），对应于触发软性 Pod 驱逐操作之前软性驱逐阈值所需持续的时间长短。 

 **生产环境如果集群管理员对集群缺乏足够的把控力，建议关闭驱逐，显示设置 --eviction-hard= 即可关闭，关闭之后驱逐相关的其他参数就不需要关注了**。

## Reserve

> `--kube-reserved`

kubernetes 系统预留的资源配置，以一组 `<资源名称>=<资源数量>` 格式表示。 （例如：`cpu=200m,memory=500Mi,ephemeral-storage=1Gi,pid='100'`）。 当前支持 `cpu`、`memory` 和用于根文件系统的 `ephemeral-storage`。

> `--qos-reserved`

【警告：Alpha 特性】设置在指定的 QoS 级别预留的 Pod 资源请求，以一组 `"资源名称=百分比"` 的形式进行设置，例如 `memory=50%`。 当前仅支持内存（memory）。要求启用 `QOSReserved` 特性门控。

> `--reserved-cpus`

用逗号分隔的一组 CPU 或 CPU 范围列表，给出为系统和 Kubernetes 保留使用的 CPU。 此列表所给出的设置优先于通过 `--system-reserved` 和 `--kube-reskube-reserved` 所保留的 CPU 个数配置。

> `--reserved-memory`

以逗号分隔的 NUMA 节点内存预留列表。（例如 `--reserved-memory 0:memory=1Gi,hugepages-1M=2Gi --reserved-memory 1:memory=2Gi`）。 每种内存类型的总和应该等于`--kube-reserved`、`--system-reserved`和`--eviction-threshold。`

> `--system-reserved`

系统预留的资源配置，以一组 `资源名称=资源数量` 的格式表示， （例如：`cpu=200m,memory=500Mi,ephemeral-storage=1Gi,pid='100'`）。 目前仅支持 `cpu` 和 `memory` 的设置。 

## Image

> `--serialize-image-pulls` default true

逐一拉取镜像。建议 *不要* 在 docker 守护进程版本低于 1.9 或启用了 Aufs 存储后端的节点上 更改默认值。

> `--registry-burst` default 10 

设置突发性镜像拉取的个数上限，仅在 `--registry-qps` 大于 0 时使用。

> `--registry-qps` default 5

如此值大于 0，可用来限制镜像仓库的 QPS 上限。设置为 0，表示不受限制。 

## Other

> `--max-open-files`  default 1000000

kubelet 进程可以打开的最大文件数量。

> `--node-status-update-frequency` default 10s

指定 kubelet 向主控节点汇报节点状态的时间间隔。注意：更改此常量时请务必谨慎， 它必须与节点控制器中的 `nodeMonitorGracePeriod` 一起使用。

> `--pod-max-pids` default -1

设置每个 Pod 中的最大进程数目。如果为 -1，则 kubelet 使用节点可分配的 PID 容量作为默认值。

> `configMapAndSecretChangeDetectionStrategy` default Watch

`configMapAndSecretChangeDetectionStrategy` 是 ConfigMap 和 Secret Manager 的运行模式。合法值包括：

- `Get`：kubelet 从 API 服务器直接取回必要的对象；
- `Cache`：kubelet 使用 TTL 缓存来管理来自 API 服务器的对象；
- `Watch`：kubelet 使用 watch 操作来观察所关心的对象的变更；

生产环境中如果存在大量使用 ConfigMap 或者 Secret 作为卷挂载到 Pod 中的场景时，Watch 策略会导致 kube-apiserver 中对应资源 Get 请求的 QPS 非常高，这里笔者也已经通过如下 PR [feat: minimize unnecessary API requests to the API server for the configmap/secret get API](https://github.com/kubernetes/kubernetes/pull/120255) 修复了这个问题并且在 v1.29 中发布，有相同使用场景而运行的 k8s 版本比较低的话，建议把此 PR pick 回来，然后调整为 Cache 策略，可以有效的降低 QPS。相关原理可以参考如下三篇：

- [High QPS for ConfigMap Get Requests - 1](https://mp.weixin.qq.com/s/bdoX-e-nRC64nOpqBwipZQ)
- [High QPS for ConfigMap Get Requests - 2](https://mp.weixin.qq.com/s/5sv2YJNWrlwmAnA8sKB5_g)
- [High QPS for ConfigMap Get Requests - 3](https://mp.weixin.qq.com/s/y6VB96Onzr6UofjIXRTKCw)

# Other

其他组件比如 kube-proxy 很多公司不使用或者用其他组件代替了，CNI，CSI，Device Plugin 等实现也众多，具体组件具体去看吧。

# 爆炸半径

## 禁止删除

可以通过 validating webhook 来设置一些规则，默认禁用一些大范围删除请求，例如删除 CRD，删除 Namespace，删除 Node，批量删除 Pod 等高危操作。这里推荐使用 [kinitiras，一款通用可编程的 admission webhook 策略引擎](https://mp.weixin.qq.com/s/P0ha0Cz5UO4rJlhVtQnQpg)来处理，对 k8s 代码没有任何侵入，想要实现什么功能，拦截什么请求，只需要添加一个对应的策略（cr 文件）即可；

## 限制性删除

### 服务端限流

kube-apiserver 开启 apf 限流，按需设置。

### 客户端限流

node 异常引起的 pod deletion，比如遇到网络问题或者其他原因导致 kubelet 异常或者无法访问 kube-apiserver 的话，如果超过一定时间，则 kube-controller-manager 就会为 node 设置对应的 taint，然后开始删除上面的容器（设置 deletionTimestamp），虽然存量 Pod 可能无法立刻被删除，但等到网络问题或者其他问题被解决，kubelet 恢复后，就会开始执行 pod 删除的操作。可以通过为 node 划分 region，zone，eviction rate 等参数进行详细的设置来控制影响范围，起到客户端限流的效果。

资源紧张时的驱逐，考虑关掉这个能力。computePodActions hash 校验导致的重启，最初这个能力正常是供 Deployment 变更时用的，比如只修改 image，现在也会用于 In-Place Update 的场景。但由于其校验逻辑是直接使用整个 container 计算 hash，在遇到集群 "原地升级/降级" 的场景时，即使没有其他任何主动变更操作，如果 container 在新旧版本的 kube-apiserver 中有字段的增减，也会导致前后 hash 不一致，从而容器重启，而这种重启虽然是单机上执行的，但波及范围是整个集群。

In-Place Update 自 v1.27 发布为 alpha 状态至今仍然是 alpha 状态，笔者也曾给社区反馈过 hash 校验导致的存量容器重启问题：[In place update trigger container restart when upgrade k8s cluster](https://github.com/kubernetes/kubernetes/issues/119187)

> 1. Old pod created by old version k8s will restart when to resize the pod spec after upgrade k8s cluster version and enable the specified feature gate
> 2. pod which have do some resize operations will restart when disable the in place update feature gate

最终这个问题被当做其成为 beta 状态的一个 PRR，可以看到即使强如社区，在遇到这种问题的时候也没有把相关场景都考虑到（确实考虑到了 hash 会变的场景，做了一定的处理，但没有考虑全），做到不影响存量容器，这是非常非常危险的事情。

要解决这个问题，只能修改 kubelet 代码，要么添加更全面的 hash 校验场景，要么在判断 hash 不一致之后额外访问全局 api，实现类似限流的能力，在获得许可后再进行实际的容器重启的动作。各有利弊，前者很有可能随着版本升级出现新的 case，需要每次升级时都关注这块逻辑，后者则是引入了外部依赖，需要考虑外部依赖异常场景。但即使实现了后者，还是需要在每次升级时关注这个问题，所以可以的话，还是尽量在根上解决问题，而不是靠限流。

# 最后

虽然怎么也飞不出，草台班子的世界，但还是可以在草台班子里面提高自己的演技。以上内容来自笔者的经验和一些狭隘的认知，如有问题或者更好的方案，欢迎一起交流~


