# 你真的搞懂 Informer 了吗？


> 代码版本 v.126

# 由来

Informer 作为 client-go 的核心，网上有众多的源码分析，原理解析相关文章，可以教给大家如何"正确"的使用 Informer。当然其前提是在 Informer 本身逻辑没问题的前提下，本篇旨在为大家指出几个 Informer 里面长期存在甚至现在仍然存在的问题。

通过这些问题可以检查下自己是否真的对工作原理理解了，而不仅仅是停留在使用的阶段，也希望引起大家注意，遇到相关问题时可以及时定位原因，解决问题，最好是可以在最初就避免问题。

下文假设大家已经看过 Informer 源码，或者看过相关源码解析的文章，因此在本文中不会过多涉及代码。

# 测验

先尝试回答几个问题，来看看自己对 Informer 的理解和掌握情况。

1. Informer vs SharedInformer 的区别，shared 的内容是什么？
2. 什么场景下要开 Resync？Informer 同步的内容是什么？
3. HasSynced 返回 true，代表什么意思？
4. 使用 NamespacedName 作为 key，有没有问题？

# 答案

1. 区别：前者只能注册单个 ResourceEventHandler，后者可以注册多个，也就是说当你需要在一个事件来了之后去触发多个动作或者有多个观察者的话，那就用后者，否则可以用前者；**shared 的是 Reflector**；
2. [需要与外部（非 k8s）的组件或者基础设施交互时使用 SharedInformer](https://github.com/kubernetes/kubernetes/issues/75495#issuecomment-475629555)，SharedInformer 同步是将 Store 里面存储的对象重新放回 Queue 中，重新触发一次全量通知，**Resync 并不会触发重新去 kube-apiserver 获取数据**；Informer 的 Resync 不会执行任何操作，直接返回；
3. 对于 Informer，HasSynced 返回 true 代表其所注册的 event handler 已经执行完了，针对 SharedInformer，则代表 **Queue（Reflector Store）里面的数据已经全部 Pop 出来，经过处理将对应的 Indexer 里面的数据分发给所有的 processorListener 了，但是否注册的所有的 event handler 已经都处理完了并不一定；**
4. 一般索引都用 ID 表示，使用 NamespacedName 会有潜在的问题，后文会举例说明；

前两个问题出现频率比较高，应该没什么问题，主要分析后两个。

# 分析

![sharedIndexInformer](informer.svg)

一图胜千言，我们按这个图来依次解释（瞎比比)上面的几个问题。

## 区别

 一个 Informer 只能通知一个观察者，而一个 SharedInformer 可以通知多个观察者。需要触发的行为通过 ResourceEventHandler 注册，对于 Informer，ResourceEventHandler 是通过参数在创建 Informer 的时候传递的，直接保存在自身结构内，而 SharedInformer 则是单独提供了方法来添加 ResourceEventHandler，保存在 SharedProcessor 内，而 SharedProcessor 保存在 SharedInformer 内。

## Shared 内容

最终发送的通知事件内容是存储在 Reflector Store 中的，也就是说为了通知多个观察者，Informer 需要有多个实例，每个实例都维护一份全量数据，而 SharedInformer 只需要一个实例，维护全量数据，有利于降低内存占用。

## Resync

本质是用 Reflector Store 也就是 DeltaFIFO 中 KnownObjects 数据同步 Queue，对应上图步骤 x。

## HasSynced

还记得之前发过的一篇设计一个分发器的内容吗？k8s 在这里巧妙的借助了两个无缓冲区的 chan 和一个 RingBuffer 实现了事件异步分发给多个观察者的能力。这块的代码值得大家学些下，可以看 client-go 的实现，也可以看这个精简版的实现：[dispatcher](https://github.com/likakuli/dispatcherdemo)。

针对 Informer 来说，HasSynced 返回 true 就是代表其注册的 ResourceEventHandler 已经处理完一遍全部数据了，而针对 SharedInformer 而言，只代表了图 7 完成，从 7 开始，后面的执行过程和前面的是异步的，所以到底是否已经被 ResourceEventHandler 处理完一遍了，未可知。

***这也就意味着你可能一直在毫不知情的错误的使用 SharedInformer。尤其针对一些在真正开始执行具体逻辑之前需要先同步处理完一遍全量数据的场景，例如调度器在真正调度 Pod 之前需要确保已经利用全量数据构建好了本地缓存，否则可能会导致错误的调度结果。***

此问题在做模拟调度项目时碰到了，社区在 [PR 113985](https://github.com/kubernetes/kubernetes/pull/113985) 修复此问题，随着 v1.27 中发布。解决方案是为每个 ResourceEventHandler 添加 HasSynced 方法，针对上述需要确保全量数据需要被处理完成一遍的场景，判断对应 ResourceEventHandler 的 HasSynced 即可。

## NamespacedName

考虑如下场景：

1. SharedInformer 收到 Pod1 的 Add 事件，并将 Pod1 添加到 Store 中；
2. SharedInformer 与 Kubernetes API 断开连接，并尝试使用 ResourceVersion1 重连；
3. Pod1 被删除后立马又被创建出来，对应 ResourceVersion2；
4. Etcd 执行了压缩，ResourceVersion2 之前的历史记录都没了；
5. SharedInformer 重新连接成功；

由于 ResourceVersion1 和 ResourceVersion2 之间数据丢失，导致 SharedInformer 得不到 Pod1 Delete 事件，在重连后可以正常看到后创建的 Pod1，此时的处理逻辑是去看下能否通过 Pod1 NamespacedName 从本地 Store 获取到对应的记录，获取到的话则认为是 Update 事件并进行分发，而正常应该将这种特殊行为判定为 Delete、Add 两个事件，因为对象的 UID 发生了变化，虽然名字没变，但已经不是之前的对象了。

这也可能会触发一些问题，为了避免这些问题，对事件类型有依赖的场景还需要在接收到 Update 事件之后，再根据新旧对象的 UID 再次判断是真正的 Update 还是 Delete 与 Add 的合并。

# 总结

历史版本中出现过众多问题，甚至一些至今仍存在的问题，有一些是细节处理，有一些是设计问题，上面分析到的也只是一小部分，其他不再一一介绍。这里介绍 Informer 是为后续做铺垫，方便理解后续的文章。我们知道大规模集群下，APIServer 会成为瓶颈，尤其在内存方面，相信很多人也遇到过类似 APIServer OOM 等问题，后续会继续分析 Watch 在客户端，服务端的实现，以及 APIServer 内存到底来自哪些部分（请求），空间复杂度如何，如何根据数据量评估内存需求，以及如何优化内存使用。敬请期待...


