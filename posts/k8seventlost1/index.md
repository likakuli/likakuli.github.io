# k8s watch 丢事件 - 1


Kube-apiserver 提供了 Watch API 来支持实时接收资源对象变化的功能，也是 Informer 实现的基础，那么我们通过 Watch 或者 Informer 本地缓存拿到的数据是否真的和 Etcd 中的数据一致呢？

先说结论：k8s watch 可能是会丢事件的。将会介绍两种丢数据的场景，分为两篇来介绍，包括原理，当前是否已经修复等信息。

注意：这里说的是丢事件，而不是丢数据。

# 现象

在 kube-apiserver 的日志中会看到 "**etcd event received with PrevKv=nil**" 的字样，资源对象被删除后在 Etcd 中已经不存在了但在 Reflector store 中仍然存在，可以在 Informer 或者 watchCache 中看到对应的对象，依赖 Informer 的组件也不会感知到资源对象被删除，通过 List API 设置 RV="0" 去 kube-apiserver 的 watchCache 中获取的话也可以看到已经被删除的对象仍然存在。

# 原理

针对类型为 DELETE 的 event，被删除的资源对象是从 event.PrevKV 中获取的。**正常情况下，delete event 的 PrevKV 不是 nil，但是在异常情况下，delete event 的 PrevKV 是 nil，导致 kube-apiserver 虽然收到了 delete event，但是他无法得知是谁被删了**。

## Etcd

为什么会返回 PrevKV 是 nil 的 delete event 呢？这个问题在社区中有对应的 [issue#prevKV not being returned if the previous KV was compacted is suprising behavior](https://github.com/etcd-io/etcd/issues/10681)。里面给出了复现步骤：

> 1. put "/x" -> "value" (revision=1)
>
> 2. create watch "/x"
> 3. delete "/x" (revision=2)
> 4. compaction
> 5. "/x deleted" watch event sent

这个问题发生的概率会比较低，第 4 步的压缩发生在 3 之前，或者 5 之后都不会有问题，只有发生在 3 和 5 之间才会有问题。kube-apiserver 默认定时触发 Etcd 压缩操作。最终 Etcd 侧修改了这种特殊场景的行为，直接返回 `ErrCompacted` 报错。

## Kube-apiserver

这个问题同时也暴露了 kube-apiserver 在处理从 Etcd 收到的 event 时存在的问题，kube-apiserver 也做了对应的功能完善，在收到 event 时，如果不是 Create event，会判断其 PrevKV 是否为 nil，是的话会报错，如下

```go
if !e.IsCreate() && e.PrevKv == nil {
		// If the previous value is nil, error. One example of how this is possible is if the previous value has been compacted already.
		return nil, fmt.Errorf("etcd event received with PrevKv=nil (key=%q, modRevision=%d, type=%s)", string(e.Kv.Key), e.Kv.ModRevision, e.Type.String())
}
```

最终上述 err 会封装为 `InternalServerError` 返会给客户端，Reflector 收到后会退出当前 ListAndWatch 的执行，开始进行下一轮的 ListAndWatch 的调用，最终通过新一轮的 List 调用就可以避免这个问题，因为这时候 List 到的数据已经不包含被删除的对象了。同时作用于 watchCache 和 Informer。

在 Etcd 通过返回 `ErrCompacted ` 来规避 PrevKV 问题后，针对这个场景，kube-apiserver 上述逻辑将不会再执行，但他仍然可以作为一个兜底的逻辑存在，因为可能还存在其他原因导致的 PrevKV = nil event 的出现。`ErrCompacted ` 会提前被拦截到，如下

```go
for wres := range wch {
		if wres.Err() != nil {
			err := wres.Err()
			// If there is an error on server (e.g. compaction), the channel will return it before closed.
			klog.Errorf("watch chan error: %v", err)
			wc.sendError(err)
			return
		}
		for _, e := range wres.Events {
			parsedEvent, err := parseEvent(e)
			if err != nil {
				klog.Errorf("watch chan error: %v", err)
				wc.sendError(err)
				return
			}
			wc.sendEvent(parsedEvent)
		}
	}
```

会先判断 wres.Err()，针对这个 case 的话会是 `ErrCompacted`，最终 kube-apiserver 会识别错误类型，如果是 `ErrCompacted` 的话，会返回给客户端 `ResourceExpired` 的报错。Reflector 在收到返回的报错后的处理逻辑与上述收到 `InternalServerError` 一样。通过 [pr#Error when etcd3 watch finds delete event with nil prevKV](https://github.com/kubernetes/kubernetes/pull/76675) 解决，在 v1.15 中发布。

# 总结

同时在 Etcd 和 k8s 侧做了能力的完善，用来保证 Etcd 在对应情况下不再返回 PrevKV=nil 的 delete event，k8s 侧也增加了兜底逻辑，即使出现 PrevKV=nil 的 event 也不会影响业务。

在系统设计时应该考虑到依赖组件可能存在的问题并尽可能添加对应的兜底策略，不能无条件的相信依赖项没有任何问题。下一篇将介绍另外一个造成丢事件的 case，仍然是和 Etcd 与 k8s 都有关系，至今仍然存在，尤其是如果使用了 v1.27 的版本的话，敬请期待~


