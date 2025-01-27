# informer relist - feature or bug?


# 由来

Informer 作为 client-go 的核心，网上有众多的源码分析，原理解析相关文章，可以教给大家如何"正确"的使用 Informer。当然其前提是在 Informer 本身逻辑没问题的前提下，本篇继续为大家指出 Informer 里面长期存在甚至现在仍然存在的问题。之前已经写过一篇来介绍，参考[你真的搞懂 informer 了吗](https://mp.weixin.qq.com/s/ciEzUs5qb9WZYMl6QC8sZA)

通过这些问题可以检查下自己是否真的对工作原理理解了，而不仅仅是停留在使用的阶段，也希望引起大家注意，遇到相关问题时可以及时定位原因，解决问题，最好是可以在最初就避免问题。

下文假设大家已经看过 Informer 源码，或者看过相关源码解析的文章，因此在本文中不会过多涉及代码。

# 万恶之源

kube-apiserver oom 的一个常见触发原因就是 informer 在 `ListWatch` 模式下触发大规模 relist 导致的，尤其是在大规模集群场景下。本篇主要介绍触发 relist 的一些原因以及其中存在的问题，`WatchList` 模式不在讨论范围内。

## Relist

首先需要明确一点，判断是否需要 relist 是客户端的行为，客户端根据执行过程中返回的报错来判断是否需要 relist。 一些典型的不需要 reslist 的场景，如**临时的网络连接问题**，**服务端限流**等。笔者曾经遇到一个 [kube-apiserver 重启导致的 relist 问题](https://mp.weixin.qq.com/s/bO5922AIU5f-dlc3NxnbWQ)，根因是 k8s 未能正确识别临时的网络连接问题，这个问题也是笔者给 k8s 提的第一个 PR。

### Too large resource version

服务端返回的报错，在传入的 RV 比 ETCD 中最大 RV 还要大，或者比 kube-apiserver Cacher 中的最大 RV 大，且 Cacher 在等待 3s 后其最新的 RV 仍然比传入的 RV 小的时候会返回此报错。后者需要开启 `ConsistentRead` 或者在 `WatchList` 模式下才会出现。前者一般不容易出现，如果遇到 Etcd Restore 时可能会出现。

### Too old resource version

服务端返回的报错，在传入的 RV 比 Cacher 中最小的 RV 还要小的时候返回此报错。在老版本中出现的频率较高，新版本中开启 watchcache 动态窗口大小后出现频率较低。另外在未开启 `ConsistentRead` 时，从一个 kube-apiserver 断开连接重新与另一个 kube-apiserver 建立连接时也可能会遇到此问题。

注意**以上两种报错触发的 relist 操作会直接穿透到 ETCD**。

### InternalError

客户端长时间连接不上服务端，或者在处理服务端返回数据时报错等。但有一个例外，为了避免 ETCD 选主失败导致 kube-apiserver 重新去 ETCD list 全量数据，特意增加了对此异常的处理，参考 [Add option to retry internal api error in reflector](https://github.com/kubernetes/kubernetes/pull/111387)，此处理目前仅仅在 kube-apiserver 访问 ETCD 时生效，在 informer 中不生效，也就是说 **informer 中的 reflector 遇到任何 InternalError 都会触发 relist。**

# 现存问题

## client-side timeout

当客户端主动关闭，或者客户端超时时，应该直接重新 watch 而不是 relist。但由于当前实现的 bug，在客户端超时时也可能会触发 relist 操作，已经提了 [PR](https://github.com/kubernetes/kubernetes/pull/129792) 来修复此问题，问题产生的原因参考此 [issue](https://github.com/kubernetes/kubernetes/issues/129683#issuecomment-2609354913)。

修复很简单，但是发现此问题的过程很坎坷。这里涉及到 watch 的实现，默认是利用 HTTP Chunked Transfer Encoding 实现的。针对 k8s 内置的资源类型，默认是通过 `lengthDelimitedFrameReader` 对 watch response body 中的 protobuf 进行 read 和 decode 来生成具体的 event。`lengthDelimitedFrameReader` 不断的从 response body 中读取内容，解码，重复这个过程，设计上如果遇到 client-side timeout 时会返回，但不会往最终的 event chan 中发送数据，但实现中有 bug 导致遇到 client-side timeout 后依然会往 event chan 写数据，从而走到上面 `InternalError` 的处理逻辑导致 relist 的出现。为什么会出现 client-side timeout 呢，因为 informer 会为 watch 添加默认超时时间，默认为 5m ~ 10m，超时后重新基于 last resource version 继续 watch。

# 总结

informer 中的 reflector 遇到任何 InternalError 都会触发 relist，这一点值得商榷。

一些设计合理的逻辑也可能会在实现中存在 bug 导致预期外的行为，有时候光看注释或者文档解决不了问题，还是得看代码实现。


