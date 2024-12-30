# Informer 内存使用优化 - 1


# 概述

Informer 几乎在所有 k8s 组件中都会用到，即使是在 kube-apiserver 中。这里主要介绍 Informer 在 v1.27 ~ 1.32.0 之间做的一些内存使用相关的优化，可以降低使用 Informer 组件的内存使用量，减少 OOM 问题的出现，提升稳定性，提升资源利用效率。

主要包含两个优化：

1. SetTransform 移除资源对象中不需要的属性；
2. ExtractListWithAlloc 解决 golang slice GC 机制导致的内存无法回收问题；

分两篇讲解，本篇主要介绍第一个优化，比较简单。

# SetTransform

Informer 提供了 SetTransform 方法，其作用是在把接收到的资源对象存储到 Reflector 之前可以做一些自定义的操作，例如修改某个属性的值。也就是说虽然 Informer 从服务端拿到的是完整的对象，但存储在本地的可以是对象的一部分。这个方法在早一些的版本（例如 1.24）中就已经存在了，但是并没有显示使用。1.29 开始在 kube-scheduler 和 kube-controller-manager 中使用此方法来优化内存使用，1.31 开始在 kube-apiserver 中使用此方法优化内存使用。

## managedFields

所有资源对象上有个共同的属性 managedFields，其存在于 ObjectMeta 中，供 SSA（Server-Side-Apply） 功能使用。此属性和 Informer 业务逻辑关系不大，几乎在所有使用 Informer 的客户端代码中都可以忽略此属性。其会随着每次对资源对象的修改更新其内容，改的属性越多，其内容也越多，甚至通过 kubectl 查看某个资源对象的 yaml 时，有一半的行数是在展示此字段的内容，可读性也非常差，好在之前的版本中 kubectl 已经默认不在 get 的默认返回内容中显示此属性，如果想显示的话可以通过 `--show-managed-fields` 参数来控制，默认是 false 即不展示此属性。

控制面适配：

[Kube-scheduler] [Improve memory usage of kube-scheduler by dropping the .metadata.managedFields field that kube-scheduler doesn't require](https://github.com/kubernetes/kubernetes/pull/119556)

[Kube-controller-manager] [Improve memory usage of kube-controller-manager by dropping the .metadata.managedFields field that kube-controller-manager doesn't require](https://github.com/kubernetes/kubernetes/pull/118455)

[Kube-apiserver] [Improved memory usage of kube-apiserver by dropping.metadata.managedFields field that self-requested informers of kube-apiserver didn't need](https://github.com/kubernetes/kubernetes/pull/124667)

# 问题修复

SetTransform 实现逻辑也引入了一些 data race 的问题，触发频率不高，甚至在某些组件内并不会触发，具体与 Informer 的使用方式有关系。目前已经修复了已知的 data race 问题，但在某些极端场景，例如自定义 Reflector 时，如果不知道某些原理可能还是会写出存在问题的代码，好在日常开发几乎不会涉及到自定义 Reflector 的需求。

涉及到的两个修复 Informer 中存在的 data race 问题的 PR 如下

[Informer] [Fixed a race condition between Run() and SetTransform() and SetWatchErrorHandler() in shared informers](https://github.com/kubernetes/kubernetes/pull/117870)

[Informer] [Fix a race condition in transforming informer happening when objects were accessed during Resync operation](https://github.com/kubernetes/kubernetes/pull/124344) 

都比较容易理解，就不再具体描述了。同时在客户端使用 SetTransform 时也做了适配，如下

[Kube-scheduler & Kube-controller-manager] [Fixed a race condition in kube-controller-manager and the scheduler, caused by a bug in the transforming informer during the Resync operation, by making the transforming function idempotent](https://github.com/kubernetes/kubernetes/pull/124352)

总结下就是客户端想对某个对象做一些属性修改时，需要判断下传进来的对象是否已经符合预期，符合的话就不需要再调用 SetTransform 来进行修改了。例如针对去掉 managedFields 属性的修改，正确的代码如下

```golang
if accessor, err := meta.Accessor(obj); err == nil {
			if accessor.GetManagedFields() != nil {
				accessor.SetManagedFields(nil)
			}
		}
```

或者

```golang
if accessor, err := meta.Accessor(obj); err == nil && accessor.GetManagedFields() != nil {
	 accessor.SetManagedFields(nil)
}
```

变相提高了对开发人员的要求，在使用此方法时需要清楚的知道其背后的逻辑和潜在的风险，以及如何写出没问题的代码。

除 k8s 项目外，如 controller-runtime 也已经做了适配，支持这些功能，也修复了上述问题。

# 总结

这个优化的核心还是减少不必要属性的存储，对于存在的 data race 的问题，除了通过锁实现外，也可以通过减少不必要的方法调用来规避问题。如果还在使用老的 client-go 版本同时又想在不升级 k8s 集群的前提下使用此优化，建议把上述 PR 都 pick 回来。

