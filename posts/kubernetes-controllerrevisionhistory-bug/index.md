# Kube-controller-manager同步数据慢


### 背景

> 版本1.12.4

线上遇到kube-controller-manager重启慢的问题，具体表现为进程重启虽然速度快，但是重启完所有数据都同步完一遍耗时很长，集群中大约5000个statefulset，在还没同步完一遍数据之前如果有statefulset的创建、删除、修改等操作，可能（和具体statefulset的操作有关，新建的情况肯定是在最后，更新和删除的情况需要看同名的statefulset是否已经被处理过了，如果是的话也会在最后处理，如果没有的话，则不会排在最后）就需要等到所有数据都同步完之后才能继续处理。

### 问题原因

#### 并发goroutine数

当前版本的statefulset controller只使用了一个goroutine来串行的处理所有的statefulset，在最新版的代码中已经支持了多个goroutine并行处理，且可配置

#### Informer List

经过添加trace信息打印每个阶段耗时，发现在根据statefulset获取其对应的controllerrevision时比较慢，如下代码段

```go
// adoptOrphanRevisions adopts any orphaned ControllerRevisions matched by set's Selector.
func (ssc *StatefulSetController) adoptOrphanRevisions(set *apps.StatefulSet) error {
  // 比较慢
	revisions, err := ssc.control.ListRevisions(set)
	...
}
```

而且同样的逻辑会在这里在执行一次

```go
// syncStatefulSet syncs a tuple of (statefulset, []*v1.Pod).
func (ssc *StatefulSetController) syncStatefulSet(set *apps.StatefulSet, pods []*v1.Pod) error {
	...
	// TODO: investigate where we mutate the set during the update as it is not obvious.
  // UpdateStatefulSet里面会再次调用ListRevisions函数
	if err := ssc.control.UpdateStatefulSet(set.DeepCopy(), pods); err != nil {
		return err
	}
	...
}

```

最终就导致同步一个statefulset就需要100+ms，总的5000+ statefulset同步完一遍就需要将近20m。

#### ControllerRevision

每个statefulset都对应一些controllerrevision资源，从字面意思就可以看出来其作用就是记录此statefulset的历史信息，这也是为什么我们可以直接对statefulset做回滚之类的操作的原因，默认情况下会为每个statefulset保留10条历史记录，在每个statefulset上有属性可配置。

其实上面说到的耗时的逻辑就是针对每个statefulset去获取孤儿controllerrevision，如果有则会领养孤儿。所以一个优化项就是直接从孤儿中找，而不是从全量中找，且把所有的controllerrevision缓存到本地，不再使用ListInformer提供的那些方法，因为这些方法始终会在全量中寻找满足条件的，而且还会用到反射，虽然数据也都在本地，但性能还是比较差的。大体思路就是在创建statesetfulset controller时同时注册controllerrevision相关的事件，把所有的revision和孤儿revision缓存到自定义的数据结构中，后续直接从里面获取即可。

### 最终效果

优化完之后最终重启一次controller-manager知道全量数据同步完一遍的耗时由20m左右缩减到1m左右，可以看到效果还是很明显的，而且还是有优化空间的，比如继续以空间换时间，在备controller-manager启动时就先把所有的数据同步完，所有更新缓存的逻辑照样执行，只是不触发其他操作，这样在主备切换时就能省掉网络传输数据的耗时，当然得衡量数据量大小，随着集群规模越来越大，master上各组件占用的内存势必越来越多，将来可能就又会面临内存不够用的情况。

### 意外收获

测试的时候发现了一个bug，孤儿controllerrevision无法被领养，且会导致statefulset同步失败，这是statefulset controller的代码bug，pr已合入master，随着v1.18版本发布。具体可参考[这里](https://github.com/kubernetes/kubernetes/pull/86801)

