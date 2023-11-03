# Kubernetes 陈年老 bug - Stale Read


# 背景

前两篇已经介绍过 Informer 和 Cacher 的实现，也介绍了其中存在的一些问题，本篇主要针对 Stale read 问题展开，分析新版 Informer & Kube-apiserver 中是如何解决这个问题的。

如果对 [Informer](https://mp.weixin.qq.com/s/ciEzUs5qb9WZYMl6QC8sZA) 和 [Kube-apiserver WatchCache](https://mp.weixin.qq.com/s/FhNl1c2BdXJK-sDb0qzahw) 还不熟悉的话，建议可以先看前两篇，或者其他有关内容讲解的文章。

# 细节

为方面描述，下文统一使用 RV 代指 Resourceversion，本节逻辑均基于 v1.26.9 版本，且忽略分页查询，因为分页是直接走 Etcd 的。

无论是 List 还是 Watch 请求，其 query 均支持传入 RV，服务端会根据请求的 RV 的不同做相应的处理，根据 RV 的值可以分为三种情况

1. 未设置或者显示设置 RV=""
2. RV = "0"
3. RV = "非 0 值"

对于前两种情况，List 会直接返回 WatchCache Store 中的内容，即服务端缓存好的 Etcd 的全部相关数据。

对于第三种情况，会等待服务端缓存数据的最大版本要超过传入的 RV 之后再返回缓存内的数据，如果等待了一段时间（3s）后缓存中的数据仍然没有达到指定版本，则会报错返回 "Too large resource version"，并告诉客户端可以在 1s 之后重试。

# 修复

## 问题分析

回顾上一篇中提到的 Stale Read 的问题

> 1. T1: StatefulSet controller creates `pod-0` (uid 1) which is scheduled to `node-1`
> 2. T2: `pod-0` is deleted as part of a rolling upgrade
> 3. `node-1` sees that `pod-0` is deleted and cleans it up, then deletes the pod in the api
> 4. The StatefulSet controller creates a second pod `pod-0` (uid 2) which is assigned to `node-2`
> 5. `node-2` sees that `pod-0` has been scheduled to it and starts `pod-0`
> 6. The kubelet on `node-1` crashes and restarts, then performs an initial list of pods scheduled to it against an API server in an HA setup (more than one API server) that is partitioned from the master (watch cache is arbitrarily delayed). The watch cache returns a list of pods from before T2
> 7. `node-1` fills its local cache with a list of pods from before T2
> 8. `node-1` starts `pod-0` (uid 1) and `node-2` is already running `pod-0` (uid 2).
>
> 详情可以参考 [issue 59848](https://github.com/kubernetes/kubernetes/issues/59848)。

问题发生在第 6 步，当客户端重启，Informer List RV="0" 时如果连接的 kube-apiserver 不是断开之前的实例，就有可能会触发这个问题。因为每个 kube-apiserver 实例的 watchCache Store 存储的数据在某个瞬间可能是不一致的（网络，机器负载等原因），而从上面的细节分析来看 List 请求在遇到 RV="0" 的时候是直接返回了 watchCache Store 内容，如果其内容是落后于断连之前的 kube-apiserver 实例的缓存的，就会导致 Stale Read 的问题，虽然最终数据会一致，但还是会出现暂时的时间回流，这对于有状态服务来说影响会比较大。

## 原理解析

此问题已经在当前 master 版本中修复，通过 ConsistentListFromCache FeatureGate 控制，开关打开后可以解决 Stale Read 的问题，默认关闭（v1.29）。

核心代码如下

```go
// GetList implements storage.Interface
func (c *Cacher) GetList(ctx context.Context, key string, opts storage.ListOptions, listObj runtime.Object) error {
	...
  
	if listRV == 0 && utilfeature.DefaultFeatureGate.Enabled(features.ConsistentListFromCache) {
		listRV, err = storage.GetCurrentResourceVersionFromStorage(ctx, c.storage, c.newListFunc, c.resourcePrefix, c.objectType.String())
		if err != nil {
			return err
		}
	}
  
  ...
  
  objs, readResourceVersion, indexUsed, err := c.listItems(ctx, listRV, key, pred, recursive)

	...
}
```

判断如果客户端传递的 RV 为前两种情况，则直接去 Etcd 中获取当前最新的 RV 作为 listRV 的值传递给最终 listItems 函数。也就是说最新版本 Informer 启动的时候虽然 List 传递了 RV="0"，但在 kube-apiserver 处理时会访问一遍 Etcd 只获取最新版本号，相当于无论客户端传递的 RV 值如何，在服务端去 watchCache Store 获取数据时，始终是携带了非 0 的 RV。

结合上面对 List 的分析，需要等后缓存中的数据达到指定版本（从 Etcd 获取到的最新 RV）后才返回，这样一来就可以保证在 List RV="0" 正常返回数据的情况下，如论连接到那个 kube-apiserver 实例，其获取到的都是最新版本的数据，从而避免 Stale Read 的产生。其代价就是多了一次 Etcd Read 请求，虽然并不需要 Etcd 返回真实数据。

这里有个细节要注意下，上面的处理只判断了 listRV == 0 的情况，**如果客户端传递过来的 RV 为第三种情况，服务端就不会再去访问 Etcd 了，这时候可能访问不同实例返回的结果仍然会出现不同的结果，但是由于服务端保证了返回的数据一定是在 listRV 之后的，也就不会出现上面 Stale Read 的问题**。

篇幅有限，将会在下一篇中介绍社区是如何消除 Informer 中 List 请求从而降低 kube-apiserver 内存使用的，以及优化后的效果，敬请关注~

