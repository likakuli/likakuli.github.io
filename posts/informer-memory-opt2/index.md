# Informer 内存使用优化 - 2


# 概述

Informer 几乎在所有 k8s 组件中都会用到，即使是在 kube-apiserver 中。这里主要介绍 Informer 在 v1.27 ~ 1.32.0 之间做的一些内存使用相关的优化，可以降低使用 Informer 组件的内存使用量，减少 OOM 问题的出现，提升稳定性，提升资源利用效率。

主要包含两个优化：

1. SetTransform 移除资源对象中不需要的属性；
2. ExtractListWithAlloc 解决 golang slice GC 机制导致的内存无法回收问题；

[上一篇](https://mp.weixin.qq.com/s/aIuERflILXhNJ2w7REe2Ug)中介绍了第一个优化，本篇介绍第二个优化。

# 原理

问题的原因和 Go GC 机制有关，slice 中只要任意元素还在被其他对象引用，slice 就无法被 GC，即使其他元素已经不再被任何对象引用了。详见 https://go.dev/blog/slices-intro#a-possible-gotcha。

## List

以 PodList 为例，Reflector 在调用 List 请求拿到 PodList 后遍历 Items 存入 store 中。

PodList 数据结构：

```golang
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
// +k8s:prerelease-lifecycle-gen:introduced=1.0

// PodList is a list of Pods.
type PodList struct {
	metav1.TypeMeta `json:",inline"`
	// Standard list metadata.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md#types-kinds
	// +optional
	metav1.ListMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// List of pods.
	// More info: https://git.k8s.io/community/contributors/devel/sig-architecture/api-conventions.md
	Items []Pod `json:"items" protobuf:"bytes,2,rep,name=items"`
}
```

store 可以简化为 map[string] *Pod  的结构。

Reflector 在获取到 PodList 后会遍历 Items，并取每个 pod 的地址，最后放到 store 中，也就是说 store 里面包含了所有 Items 的引用。这个时候问题就来了，比如 Informer 初始化时有 10000 个 Pod，运行过程中删除了 9999 个，只剩一个 Pod，但这 9999 个 Pod 对应的内存无法被 GC 回收，原因在开头已经说了。

# 解决

有两种解决方式，一种是解决 slice 无法被 GC 的问题，另一种是直接规避问题。

## GC

让 store 中存储的对象不再引用 Items 就可以解决此问题。有两种方式：

1. 对 Items 中的每个对象执行 deep copy 后存入 store；
2. 对 Items 中的每个对象执行 shadow copy 后存在 store；

在内存使用来看，如果采用 deep copy，会导致在处理数据时存在内存 double 的风险，而且执行 deep copy 耗时也相对较长。最终采用的是 shadow copy 的方案，即对所有 Items 对象做浅拷贝后再取地址，保存到 store 中。浅拷贝需要申请的内存量相比深拷贝要少很多，基本是一些基础类型的属性，或者 slice 结构本身，而无需复制 Data。

无论哪种方式都不可避免的需要申请内存，一定程度上导致内存使用量和执行耗时的提升，这是不可避免的，幸运的是在解决此问题的过程中发现了 Reflector 代码实现中使用 Reflect 时的一些可以优化的地方，而这些优化恰好可以抹平浅拷贝带来的影响。

涉及到的所有的改动在一个 PR 中完成，分多个 commit，非常方便阅读。具体详见 [Faster ExtractList. Add ExtractListWithAlloc variant](https://github.com/kubernetes/kubernetes/pull/113362)。

为了方便大家理解，这里给出一段问题复现和解决的示例程序，参考 [SliceGC](https://github.com/likakuli/SliceGC)。

此方式需要升级所有使用了 Informer 的程序。

## 规避

由于是在 List 时引入的问题，如果不执行 List 也就没问题了。恰好新版本中支持了 client-go 通过 feature-gate 来控制是否开启 WatchList，直到写本文时最新的 v1.32.0 版本，此特性开关仍然是默认关闭状态。

此方式除升级所有使用了 Informer 的程序外，还依赖于升级 k8s 集群，同时依赖于升级 ETCD 集群，因为 WatchList 对 ETCD 功能有依赖。

