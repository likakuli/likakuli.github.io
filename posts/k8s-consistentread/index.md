# k8s List 一致性读


# 历史文章

[k8s stale read](https://mp.weixin.qq.com/s/SMnKpnP5J07HBeG35oC9WQ)

[ListWatch 到 WatchList](https://mp.weixin.qq.com/s/futHT0njb5y2UHLeg7BL7w)

# 原理

一致性读按严格程度划分，k8s 支持的顺序一致性，而 etcd 支持的是更严格的线性一致性。详情可以参考这篇文章 https://fgiesen.wordpress.com/2014/07/07/cache-coherency/。

之前分享了 k8s stale read 的问题，核心就是从 **kube-apiserver watch cache** 返回数据的时候一致性由从 etcd 返回数据时的**线性一致性降低成了顺序一致性**。问题在 watch cache 引入时就存在一直到了现在。

## 社区实现

涉及到两个 KEP，[3157-watch-list](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/3157-watch-list) 和 [2340-consistent-read-from-cache](https://github.com/kubernetes/enhancements/tree/master/keps/sig-api-machinery/2340-Consistent-reads-from-cache)，以后者为主，后者用到了前者实现的部分函数。

核心逻辑如下，这里就不翻译了，保持原汁原味。

> When a consistent LIST request is received and the watch cache is enabled:
>
> - Get the current revision from etcd for the resource type being served.
> - Check if the watch cache already has the current revision, if not:
>   - Add a "waiting read" and notify the goroutine running in background.
>   - Wait for watch cache to catch up. If wait times out, reject the request. (see "What if the watch cache is stale?" section for details)
>   - Remove a "waiting read"
> - Serve the request from watch cache
>
> In background, run a dedicated goroutine that will:
>
> - Wait for changes to "waiting read" count.
> - Repeat as long there is at least one "waiting read":
>   - Send `WatchProgressRequest` request to etcd.
>   - Wait 100ms
>
> Consistent GET requests will continue to be served directly from etcd. We will only serve consistent LIST requests from cache.

简单说就是当收到 LIST 请求并且满足 consistent read 条件时，会先调用 `getCurrentResourceVersionFromStorage` 函数（第一个 KEP 里面实现的）获取 Etcd 最新的 RV，记为 RV1，watch cache 开始 watch Etcd 并将收到的 event 的 RV 记为 RV2，如果 RV2 追上了 RV1，则认为缓存中是最新数据了，然后把对应的数据集返回给客户端。

后台 goroutine 存在的原因在于 watch cache 是资源级别（如 Pod，Node 对应各自的 watch cache）的，而 etcd RV 是全局的。etcd 已经支持了定期（默认 10m，公有云上一版设置为 5s）给所有客户端发送一次 ProgressNotify event 来通知客户端此时服务端的最大 RV 值，客户端收到此类事件后会更新本地 RV 值，所以理论上 watch cache 最大 RV 与 etcd 实际最大 RV 的最大延迟就是这个定期同步的周期。因为上面的逻辑是需要等 watch cache 数据已经追上 etcd 数据后才开始返回，显然默认 10m 的数据延迟对于 List 请求是不可接受的。

所以通过每 100ms 由客户端主动请求一次 ProgressNotify，让 etcd 返回最新的 RV，这样就可以把延迟缩小到 100ms 以内，比如 List 全量数据如果需要 20s 的话，总的最大耗时就是 20.1s（100ms + 20s）。相比开启 consistent read 之前，List 的总耗时会略有增加。

## 当前进展

上面提到了满足 consistent read 条件时，才会启用这个能力。那么条件是什么呢？

1. 开启了 `ConsistentListFromCache` 的 feature gate
2. Etcd 版本在 v3.4.31+ 或者 v3.5.13+
3. List 请求的 ResourceVeresion 为空

只有满足上述三个条件才会命中 consistent read 的逻辑。其中条件 3 最初的实现是针对 RV="0" 和 RV="" 的 List 请求都会生效，最近通过 [Fix enabling consistent list from watch cache also works for resourceVersion=0](https://github.com/kubernetes/kubernetes/pull/123676) 修改为当前的状态，经过和社区沟通，以及从代码，KEP 确认，发现设计之初的目标确实只是针对 RV="" 生效。影响主要体现为目前使用 Informer 发起的 List 请求默认都是 RV="0"，也就是说针对存量 informer 来说，consistent read 是无效的，除非开启 WatchList 的功能，改用 Watch 代替 List。

## 相关问题

consistent read 依赖 Etcd 的版本，其实 Etcd 在很早就已经支持了 `WatchProgressRequest` 了，但是在使用过程中发现有 bug，直到上面提到的版本才修复，所以加了对 etcd 版本的判断逻辑。

## 问题一

[Requested watcher progress notifications are not synchronised with stream](https://github.com/etcd-io/etcd/issues/15220)

表现为客户端实际还没有收到指定 RV 的数据，但已经收到了指定 RV 的 ProgressNotify event。原因在于存在两个途径传递数据，一个是 `sws.ctrlStream`，另一个是 `sws.watchStream`，后者用来发送携带有效荷载的 event，前者发送 ProgressNotify，所以导致可能在后者尚未返回给客户端时，如果此时前者有数据的话会被返回给客户端，给客户端一个错觉，自己已经拿到最新数据了，但实际上并没有。

代码如下，删减了无关代码

```golang
func (sws *serverWatchStream) recvLoop() error {
	for {
		...
    
		switch uv := req.RequestUnion.(type) {
		case *pb.WatchRequest_CreateRequest:
			...
      id, err := sws.watchStream.Watch(mvcc.WatchID(creq.WatchId), creq.Key, creq.RangeEnd, rev, filters...)
      ...
		case *pb.WatchRequest_CancelRequest:
			...
		case *pb.WatchRequest_ProgressRequest:
			if uv.ProgressRequest != nil {
				sws.ctrlStream <- &pb.WatchResponse{
					Header:  sws.newResponseHeader(sws.watchStream.Rev()),
					WatchId: -1, // response is not associated with any WatchId and will be broadcast to all watch channels
				}
			}
		default:
			// we probably should not shutdown the entire stream when
			// receive an valid command.
			// so just do nothing instead.
			continue
		}
	}
}

func (sws *serverWatchStream) sendLoop() {
	...

	for {
		select {
		case wresp, ok := <-sws.watchStream.Chan():
			...
			if !fragmented && !ok {
				serr = sws.gRPCStream.Send(wr)
			} else {
				serr = sendFragments(wr, sws.maxRequestBytes, sws.gRPCStream.Send)
			}
			...

		case c, ok := <-sws.ctrlStream:
			...
			if err := sws.gRPCStream.Send(c); err != nil {
				...
			}

			...
		case <-progressTicker.C:
			sws.mu.Lock()
			for id, ok := range sws.progress {
				if ok {
					sws.watchStream.RequestProgress(id)
				}
				sws.progress[id] = true
			}
			sws.mu.Unlock()

		case <-sws.closec:
			return
		}
	}
}
```

recvLoop 获取数据并写到对应的 chan 中，sendLoop 消费对应的 chan，最终把结果写到 response 中。因为同时往上述提到的两个 chan 中写数据，消费方就会出现顺序错乱的问题。

存在两种发送 ProgressNotify 的方式，一种是 server 端定时触发，一种是客户端主动请求。此 bug 只有在客户端主动请求时才会出现。针对 server 端轮询的方式，本身就已经是通过 `sws.watchStream` 发送 ProgressNotify，所以并没有并发问题。这是一个很神奇的地方，可能是因为不是一个人实现的才导致出现两种实现方式。

通过 [etcdserver: Send requested progress notifications through watchStream](https://github.com/etcd-io/etcd/pull/15237) 修复此 bug，核心逻辑是 ProgressNotify event 也通过 `sws.watchStream` 传递，但是需要等待有效荷载 event 同步完成后再发送。

## 问题二

[Etcd not respond to progress notification on newly created watch RPC until there is a event in a watch](https://github.com/etcd-io/etcd/issues/17507)

此问题是修复问题一的 PR 引入的新问题，现象是对于新创建的 watch 请求，如果其携带了一个较老的 RV，这个 watch 会被认为是 unsynced。如果没有与之匹配的真实的 event 发生的话，即使这个 watch 在被认为是 synced 状态后，Etcd 也**无法给所有 watcher 发送 ProgressNotify 事件**，直到有新的事件到来后才可以正常开始发送 ProgressNotify。

这是个非常有意思的问题，涉及到很多细节。

1. 有一个后台 goroutine `syncWatchers`，每 100ms 执行一次，尝试同步未同步完成的 watcher 并更新 synced 和 unsynced 维护的 watcher 列表；
2. 新创建的 watcher 会根据其 RV 判断是否为 synced 状态，如果 RV=0 或者 RV > s.store.currentRev 则会被认为是 synced，否则认为是 unsynced；
3. k8s reflector 与 etcd 交互时，存在发起 RV < s.store.currentRev 的请求，这些请求对应的 watcher 在 etcd 看来是 unsynced 状态；
4. 问题一的 PR 实现逻辑
   1. recvLoop 里面接收到客户端主动请求 `WatchRequest_ProgressRequest` 后判断变量 `deferredProgress` 如果为 false，则尝试调用 `sws.watchStream.RequestProgressAll`，此函数会先判断是否所有 watcher 都已经是 synced 状态，如果不是则直接返回 false，如果是则给所有 watcher 发送 ProgressNotify event。如果存在 unsynced 状态的 watcher，则设置 `deferredProgress` 为 true；
   2. sendLoop 里面在从 `sws.watchStream` 收到有效荷载事件并发送完成后会判断 `deferredProgress` 是否为 true，是的话会再次尝试调用 `sws.watchStream.RequestProgressAll` 试图在所有 watcher 都是 synced 状态时为其发送 ProgressNotify 事件，发送成功的话则设置 `deferredProgress` 为 false；

问题产生的原因和以上四个逻辑有关

1. T1 时刻客户端发起了 watch 请求，RV < s.store.currentRev，被认为是 unsynced 状态；
2. T2 时刻客户端主动请求 ProgressNotify，etcd 收到请求后命中 4.1 的逻辑中，由于存在 unsynced 的 watcher，所以不会给任何 watcher 发送 ProgressNotify event，`deferredProgress` 最终被设置为 true；
3. T3 时刻后台的 `syncWatchers` 发现刚才的 watcher 并没有需要发送的数据，会将其从 unsynced 列表中删除，添加到 synced 列表中；
4. 如果 T3 之后的某个时间有满足此 watcher 的有效荷载的 event 发生，从而走到 4.2 的逻辑里面，此时因为 `deferredProgress` 是 true，并且 watcher 已经变成了 synced 状态，所以 ProgressNotify 可以发送成功；但是**如果在 T3 之后一直没有满足条件事件发生，也就无法走到 4.2 里面，即使此时客户端（kube-apiserver）过了 100ms（k8s 里面的设置） 又去重新主动请求 ProgressNotify 事件，etcd 收到请求后走到 4.1 的逻辑，但由于此时 `deferredProgress` 仍然是 true，所以仍然不会去请求 `sws.watchStream.RequestProgressAll`，最终就会导致在没有满足条件的 event 发生之前，所有的 watcher 都不会受到 ProgressNotify event 了，这是一个非常严重的问题，因为 consistent read 严重依赖于其返回的结果来判断是否 watch cache 已经追上了 etcd，如果一直没有响应，则 consistent read 的 List 请求将会一直等待直到超时；**

此问题通过 [Fix progress notification for watch that doesn't get any events](https://github.com/etcd-io/etcd/pull/17557) 被解决。

其实在 etcd 看来，如果没有 event 要同步，那么就不需要发送 ProgressNotify，这是社区给的回复，设计如此。但在 k8s 看来，这显然是不可接受的，最后 etcd 社区还是接受了这个修改，但是目前看相关的一些文档和注释并未同步更新。

# 总结

无论是要开启 WatchList 还是 Consistent Read，Etcd 版本需要满足在 v3.4.31+ 或者 v3.5.13+。Consistent Read 目前尚未支持全部 List 请求，后续还会有进一步的实现。

慎用 alpha 的功能，一直在变化。就像之前提到的 watch RV=0 的请求一样，从最初的从缓存获取数据并返回，改为穿透到 etcd，现在又被改了回来，加了个 feature gate，默认还是从缓存中返回。


