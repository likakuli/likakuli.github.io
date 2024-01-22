# k8s 越来越复杂了吗？


> 从一个看似简单的功能说明 k8s 确实越来越复杂了。

# 约定

k8s list 返回结果中的元素集合是按照字母顺序从 a 到 z 升序排列的。

# 原因

这个约定存在的原因是为了保持开启 WatchCache 功能前后 list 请求返回结果的一致性。在关闭 WatchCache 功能的情况下，请求直接透传给 Etcd，此时返回的结果就是字母升序排列的。

# 实现

## Etcd

在关闭 WatchCache 时 list 结果有序，这个能力是 Etcd 提供的。kube-apiserver 的实现中调用 Etcd API 执行 Range 操作的时候并没有显示指定 SortOrder 和 SortTarget，但最终却返回了按照 key 排序的数据。如果直接使用 Etcdctl 去获取指定 key collection 的话，不需要显示指定顺序，返回的结果也是有序的。

这就涉及到 Etcd Range 的实现，在不显示设置排序顺序和排序对象的时候，默认返回 key 按照字母升序排序后的结果，相关的代码如下

```go
// 最终排序位置
func (ti *treeIndex) visit(key, end []byte, f func(ki *keyIndex) bool) {
	keyi, endi := &keyIndex{key: key}, &keyIndex{key: end}

	ti.RLock()
	defer ti.RUnlock()

	ti.tree.AscendGreaterOrEqual(keyi, func(item btree.Item) bool {
		if len(endi.key) > 0 && !item.Less(endi) {
			return false
		}
		if !f(item.(*keyIndex)) {
			return false
		}
		return true
	})
}
```

如果是沿着调用链路一步步看的话，在前一步（默认线性一致性读，需要等待 appliedIndex 赶上 confirmedIndex）返回数据时的代码中也有相关注释

```go
func (a *applierV3backend) Range(ctx context.Context, txn mvcc.TxnRead, r *pb.RangeRequest) (*pb.RangeResponse, error) {
	...

	sortOrder := r.SortOrder
	if r.SortTarget != pb.RangeRequest_KEY && sortOrder == pb.RangeRequest_NONE {
		// Since current mvcc.Range implementation returns results
		// sorted by keys in lexiographically ascending order,
		// sort ASCEND by default only when target is not 'KEY'
		sortOrder = pb.RangeRequest_ASCEND
	}
	...
}
```

注释写的很明显了，默认情况下 mvcc.Range 就是返回按字母升序排序后的结果。

## WatchCache

如果在开启 WatchCache 的情况下调用 list 可能会发现结果并不是有序的，这是因为直到 v1.27 才开始加上有序的功能，换句话说就是从有了 WatchCache 开始直到 v1.27，list 都不是严格有序的，也就是没有遵循之前的约定。如果你的客户端依赖 list 结果的顺序，那么在使用 v1.27 之前的版本的时候可能就会遇到问题了，如果没有遇到问题，说明客户端对顺序没有依赖，而这也是大部分的场景，也正如此，这个问题才不容易被发现。

List 返回的是 WatchCache store 中的数据，而 store 中的数据是无序的。在 v1.27 [Reuse generic GetList test for watchcache and fix inconsistency issues for both etcd3 and watchcache](https://github.com/kubernetes/kubernetes/pull/113730) 开始在从 store 返回最终数据时进行了主动排序的操作，原理很简单，就是实现了 golang sort.Interface 接口，然后调用了 sort.Sort 方法对 slice 进行排序。

看似人畜无害的一个行为，引入了一个新的问题，sort.Sort 的执行是在加锁的情况下执行的，而在 Reflector 处理每个从 Etcd 返回的 event 的时候也会进行加锁操作，因此 list 操作就会对 event 的处理产生影响，导致一些不必要的处理延迟。而这个问题最终在 v1.30（尚未发布） [Don't sort in the critical section](https://github.com/kubernetes/kubernetes/pull/122027) 才得到修复，通过 defer 把排序操作放在了加锁前面。

即使在没有上述问题的情况下，针对每次请求都额外多一次排序操作是否会对 API 延迟或 kube-apiserver 内存有一些影响呢？或者影响有多大呢？这就和其使用的排序算法有关了，上面提到使用 sort.Sort 进行排序，不同版本性能不一样，最新的实现是 [pdqsort](https://blog.csdn.net/ByteDanceTech/article/details/124464192) 算法，是字节贡献给社区的，其性能是之前的 2~60 倍，无需额外内存。所以对最终的内存没有影响，对延迟的影响取决于数据量的大小，但相比于对数据进行序列化，网络传输等的耗时，此处排序的耗时显得微乎其微。如果要较真的话，这里还可以再修改一下，改成使用 slices.Sort 的方式，使用泛型 + pdqsort，耗时会更低。

那有没有更好的办法来实现返回有序的效果呢，能想到的一种方案是在处理 event 将资源对象保存到 WatchCache store 的时候就保持 store 有序，这样可以避免每次 list 时的实时排序操作。但收益如何需要实现之后对比评估，理论上收益并不大。

## WatchList

WatchCache 是将 store 中的数据排序后一次性的返回给客户端，WatchList 为了优化 kube-apiserver 内存消耗，改用流的方式实现 list 的效果，参考 [从 ListWatch 到 WatchList](https://mp.weixin.qq.com/s/futHT0njb5y2UHLeg7BL7w)，那么他返回的结果也应该遵循规范做到按字母升序排列。

在最新的 v1.29.0 实现中 WatchList 还是 alpha 状态，尚未做到严格有序，原因是 WatchList 的实现是用 watchCacheInterval 的数据（WatchCache store 或者 cyclic buffer 中的数据） + cacheWatcher input chan 中的数据作为 list 的结果，由于 cacheWatcher input chan 中的数据是按 resourceversion 排序的，而且必须是按 RV 排序，就会导致最终的数据无法严格字母升序。

社区也是在对此功能进行开发，目前存在两个尚未合并的 PR，[Ensure that initial events are sorted for WatchList](https://github.com/kubernetes/kubernetes/pull/120897/) 和 [storage/cacher: ensure the cache is at the Most Recent ResourceVersion when streaming was requested](https://github.com/kubernetes/kubernetes/pull/122830)。前者是对前半部分 watchCacheInterval 的数据进行排序，后者是用来保证 watchCacheInterval 中的数据已经是 list 所需的全量数据，也就是说不再需要从 cacheWatcher input chan 拼接数据了。

理论上通过上面两个改动是可以实现 WatchList 下的 list 有序的，但同样会引入新的问题，即当前的实现是服务端收到 Watch 请求后会立马开始往客户端发送数据，而上述改动则需要服务端等待 watchCacheInterval 是全量数据，排序后才开始往客户端发送数据，这就引入了一个数据处理的延迟。也就是说处理方式变成了从当前的有一条数据就发一条的实现，变成了要先等到服务端是全量数据后再开始一条一条的发送给客户端。

那么这个延迟会有多大呢，这个值跟 Etcd 的配置有关，当前的实现是依赖 ProgressNotify 的，通过 `--experimental-watch-progress-notify-interval` 控制 ProgressNotify event 发送的周期，默认是 10m，也就是说默认情况下会存在最大 10m + 1.25s 的延迟，这显然是不可接受的。可以通过缩小此参数的值来缩短延迟，一般情况下设置为 5s，也就会有最大 5s + 1.25s 的延迟。这个延迟其实也很难让人接受，其中的 1.25s 是来自 bookmarktimer 的 1s~1.25s 的定时周期导致的。社区计划是利用 ConsistentRead 机制中已经使用到的 RequestProgressNotify 机制，由 kube-apiserver 周期性（100ms）主动请求 Etcd 发送 ProgessNotify 的原理把这个周期缩短到 100ms（1.25是否存在待定），此功能已经在 ConsistentRead FeatureGate 开启后的 List 请求中使用到了，尚未在 WatchList 中支持。

这个延迟和 ConsistentRead/WatchList + ProgressNotify 机制有关，ConsistentRead/WatchList 都是先请求 Etcd 获取当前最大的 RV，等待 Cache 数据追上 RV 之后才开始后续流程的，问题就在这个最大 RV，因为 Etcd 的 RV 是全局的，由于节点一直在上报 lease 状态，就会导致 Etcd 中的最新 RV 一直在增加，但是资源对象的 RV 很可能是一直不变的。例如获取 default namespace 下的所有 pods，即使在此 ns 下没有任何 pod 的情况下使用 ConsistentRead 或者 WatchList 时也能感觉到明显的延迟，延迟的大小和上面提到的 Etcd 参数有关，如果没有显示指定的话给人的感觉就是一直无法返回数据直到超时或者取消请求，避险问题，感兴趣的话可以用最新版本（v1.29.0）测试下。

# 结束语

一个看似简单的 list 有序，竟也会牵扯到这么多的内容，而且还有很多尚未实现的功能。k8s 为了一个基本不会用到的 list 有序，需要做这么多的工作，复杂度的提升可想而知，谁知道里面还会有多少这种实际几乎不会用到的功能呢？！

