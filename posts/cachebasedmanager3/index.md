# high QPS for configmap GET requests in kube-apiserver - 3


# 背景

线上 k8s 集群 kube-apiserver 的 ConfigMap Get 操作 QPS 较高，且同时间段 Etcd 中 ConfigMap 资源的 Get 操作 QPS 也较高，看日志多数请求的发起方是 kubelet。对应 k8s v1.22.13 版本代码，同时在 v1.28.0 测试现象相同。kube-apiserver 日志大致如下：

```shell
2023-08-23T08:55:54.331196195Z stderr F I0823 08:55:54.330840       1 httplog.go:132] "HTTP" verb="GET" URI="/api/v1/namespaces/default/configmaps/nginx-cfgmap" latency="1.926865ms" userAgent="kubelet/v1.28.0 (linux/amd64) kubernetes/855e7c4" audit-ID="36cfcbe3-d76a-4a4d-b251-47cc2df060cb" srcIP="192.168.228.2:59052" apf_pl="system" apf_fs="system-nodes" apf_iseats=1 apf_fseats=0 apf_additionalLatency="0s" apf_execution_time="1.269527ms" resp=200
2023-08-23T08:57:09.333913507Z stderr F I0823 08:57:09.333470       1 httplog.go:132] "HTTP" verb="GET" URI="/api/v1/namespaces/default/configmaps/nginx-cfgmap" latency="1.810334ms" userAgent="kubelet/v1.28.0 (linux/amd64) kubernetes/855e7c4" audit-ID="563bd337-df29-4342-afd0-9ca6e0632f0f" srcIP="192.168.228.2:59052" apf_pl="system" apf_fs="system-nodes" apf_iseats=1 apf_fseats=0 apf_additionalLatency="0s" apf_execution_time="1.177012ms" resp=200
2023-08-23T08:58:14.338971779Z stderr F I0823 08:58:14.338630       1 httplog.go:132] "HTTP" verb="GET" URI="/api/v1/namespaces/default/configmaps/nginx-cfgmap" latency="1.563356ms" userAgent="kubelet/v1.28.0 (linux/amd64) kubernetes/855e7c4" audit-ID="45350dc7-7a4b-43f1-8972-3b8053578234" srcIP="192.168.228.2:59052" apf_pl="system" apf_fs="system-nodes" apf_iseats=1 apf_fseats=0 apf_additionalLatency="0s" apf_execution_time="929.214µs" resp=200
```

# 由来

定位此问题的过程中花了一定的时间，同时也纠正了一些有关 kubelet 内 Pod 处理的错误理解。本篇旨在描述上述现象产生的原因及潜在问题，同时也希望能帮助大家更进一步的理解 kubelet 对 Pod 的处理逻辑。

由于涉及到的逻辑较多，因此将拆分成三篇来写：

1. ConfigMap Get 请求的来源？
2. 为什么 QPS 高？为什么没有走 kube-apiserver 缓存？
3. 问题如何解决？

本篇主要介绍问题如何解决。

# 回顾

前两篇内容总结对应这个图，分三块：syncPod、dswp（disiredStateOfWorldPolulator）、volumemanager。

![image-20230824154157569](allinone.png)

接下来描述一下整个流程：

1. kubelet 主 goroutine 启动定时器（图左上）每 1s 尝试从 queue 里面获取需要同步的 pod，启动一个专用 goroutine （图左下1）负责 dswp populatorLoop 每 100ms 一次的定时执行，启动一个专用 goroutine （图左下2）负责 volumemanager reconciler 每 100ms 一次的定时执行，所述三者完全并行；
2. 新 pod 创建后 kubelet 为其分配自己专用的 podworker goroutine，podworker 启动后进入无限循环直到 pod 被删后 kubelet 才会去清理这个 worker，worker 收到有新 pod 需要创建的请求后会去执行 `syncPod` （图右上）操作，这里主要关注三个动作：
   1. RegisterPod：他会最终标记本地 ConfigMap 缓存无效，记为 t1 时刻；
   2. WaitForAttachAndMount：他会把 pod 从 dsp 的 processedPods 数据中踢出，记为 t2 时刻；
3. `syncPod` 执行完之后会执行 `completeWork` （图深黄色部分），会重新把 pod 入队列，并基于 `--sync-frequncy` 设置一个有效时间，时间到了之后才能被 1 从 queue 里面获取到；
4. dswp populatorLoop 触发后会判断 pod 是否处理过，处理过直接 return，没有处理过则把 pod 和 volume 信息添加到 dsw（disiredStateOfWorld） 的 volumesToMount 中，并设置 remountRequired 为 true；
5. volumemanager reconcile 触发后先判断是否未挂载或者需要重新挂载，需要的话会获取要挂载的 ConfigMap 信息，缓存无效则直接去 apiserver 最终请求 etcd 最新数据，有效但超时则去 apiserver 获取但不走 etcd 直接返回 apiserver 缓存结果，未超时则直接返回自己本地缓存的数据不再请求 apiserver，最后用获取到的信息进行 mount 操作，成功后设置 remountRequired 为 false；
6. 1 中的定时器在一段时间（3 中 enqueue 入队列计算的时间）后再次走到 pod worker 走一遍 2，3 的逻辑，同时也会触发 4，5 的再次执行；

# 分析

有一个问题是比较明显的：从日志看最终都是穿透到 etcd 去了。为什么这里要区分开针对第一次请求要去 etcd 获取，而真多后续因缓存过期导致的请求就可以直接从 apiserver cache 返回了？注释里面写了特意这么设计，也即是说设计如此。本人最初想法是不要区别对待两种请求，都直接从 apiserver cache 返回即可，因为另一种 watchBasedManager 是基于 watch 实现的，reflector 在第一次执行时就是直接从 apiserver 缓存获取数据而不会走到 etcd。这样的话就可以保持两者一致的行为：本地缓存失效后始终从 apiserver 返回，不再穿透到 etcd。但经过与社区讨论，他们觉得还是要保持当前实现，即第一次还是要从 apiserver 获取并穿透到 etcd，目的是获取最新的数据。

虽然统一不走 etcd 的方案被驳回，但确实方案基本只是降低了 etcd 请求，并没有缓解 apiserver 的请求，因为每次都还是会标记缓存失效再去请求 aspiserver 的。

社区也给出了另外一种解法，即不要在 `AddReference` 时每次都标记缓存失效，而是只有在第一次时标记缓存无效。这明显是要比之前的方案好，因为这样不止能降低 etcd 的请求，还能降低 apiserver 的请求，更大限度的利用本地缓存。

针对规模较小的集群，效果 apiserver qps 优化并不明显，针对大规模集群的话则比较明显。因为 kube-controller-manager 会根据节点数量通过为每个节点设置 annotation：`node.alpha.kubernetes.io/ttl` 来控制每个节点本地缓存有效期，规则如下

```go
ttlBoundaries = []ttlBoundary{
		{sizeMin: 0, sizeMax: 100, ttlSeconds: 0},
		{sizeMin: 90, sizeMax: 500, ttlSeconds: 15},
		{sizeMin: 450, sizeMax: 1000, ttlSeconds: 30},
		{sizeMin: 900, sizeMax: 2000, ttlSeconds: 60},
		{sizeMin: 1800, sizeMax: 10000, ttlSeconds: 300},
		{sizeMin: 9000, sizeMax: math.MaxInt32, ttlSeconds: 600},
	}
```

size 对应节点数量，ttlSeconds 代表 ttl 时长，0 代表使用默认值，默认值在 kubelet 里面配置的 1min。

# 效果

上述修改方案会带来一个原理上的改变，改之前可以通过 `--sync-frequency` 控制同步周期，进而实现重新获取 ConfigMap 的效果，即这个参数可控制 ConfigMap 修改后在容器内看到变化所需要的时间长短，最长不会超过 1.5 * `--sync-frequency` 的时间。

修改后这个参数就不再具备此作用，何时可以生效完全依赖上述所讲 ttl，只有过期后才会重新去 apiserver 获取。而 ttl 又受集群规模影响，小于 100 台的集群，1m 失效，也就是每分钟获取一次，大于 9000 机器的集群，每 10min 获取一次，由此可见集群规模越大，修改后的效果越明显。拿 5000 节点集群距离，之前需要每 1 ~ 1.5min 获取一次，修改后需要 5m 获取一次，QPS 降低为原来 1/5。这还只是针对 apiserver 来说，针对 etcd 来说的话，qps 直接降成常数 1，因为之后第一次需要访问 etcd，综合评估下来效果显著。

# 总结

经过这个系列的三篇内容，我们详细梳理了 ConfigMap 作为 Volume 挂载的整个逻辑和流程，找到了问题，也进行了优化。理清问题本身（结果）固然重要，定位问题的思路和态度（过程）也很重要，比如需要有一颗刨根问题的心，也需要有耐心，沉住气，保持对技术的敏感，比如为什么代码逻辑里面有设置 resourceversion=0，但还是没有走缓存这种细小问题，如果一晃而过，可能也就不会有最终问题的高效优雅的解决。问题的解决大部分的精力都是在分析分体当中，而最终代码的改动可能只占一小部分的精力，当然这里说的是针对解决问题的场景，而针对实现新功能的场景，可能就又不一样了。

如果对具体修改感兴趣的话，可以参考 https://github.com/kubernetes/kubernetes/pull/120255，可以看到相较于整体的分析来看，代码的改动很少，但精髓应该还是分析问题的过程，培养分析和解决问题的能力要比知道问题最终的解决途径更值得学习。授人以鱼不如授人以渔，提升自己。


