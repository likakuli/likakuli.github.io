# labelselector 2则性能优化


# 背景

近期在定位 InterPodAffinity 插件的性能问题，部分问题已经在 1.33 中修复，比如 [Improve Goroutines metric calls in parallelizer.Until](https://github.com/kubernetes/kubernetes/pull/128999)，除此之外还发现一些其他尚未修复的问题，并提了对应修复的 PR，这里做个分享。

# Selector

Selector 是 k8s 里面非常常见的数据结构，经常用来根据 label 筛选对象。以以下 selector 为例：

```yaml
selector:
  matchExpressions:
    - { key: tier, operator: In, values: [cache, rds] }
```

其筛选逻辑是在 labels 中查找 tier 是否存在，不存在则返回 false，存在的话获取对应的 value，然后判断 value 是否在 [cache, rds] 中。乍一看这么实现也没有问题，但实际上当前的实现访问了两遍 map，第一遍访问只判断 key 是否存在，第二遍访问只返回 value。在 go 里面，完全可以只访问一遍 map 就可以实现上述效果，即经典的 map 访问表达式：val, ok := map[key] 的形式，既能获取知否存在，也能拿到其 value。

访问两遍和访问一遍的在性能上会有很大的差距吗？map 不是 O(1) 的时间复杂度吗？如果只是调用一次 Match 方法，那么确实可以忽略这个差距，如果要调用很多遍的话，这个差距就是不可忽视的，尤其是在 InterPodAffinity 这种需要根据 Topology 进行大量计算的场景下，在集群规模比较大，且集群中存在大量携带了 Preferred PodAffinity 的 Pod 时，这个方法在一次调度周期内可能被执行百万次。如果用 pprof 看的话，可以看到有个热点在 mapaccess2_faststr 函数中，此函数是 golang runtime 内置的函数，在无法修改其实现的前提下，降低对其的调用是一个缓解期性能问题的方法。已经通 PR [optimize label selector match performance](https://github.com/kubernetes/kubernetes/pull/132232) 修复，合入 master。

# List

List 时经常会传递 selector 对象进去用来筛选对象，我们可能会常用到 labels.Everything() 对象用来返回所有的结果。labels 内部还内置了另一个 labels.Nothing() 的对象，与前者恰好相反，表示不需要返回任何对象。但是，List 最内层的实现并没有判断传入的 selector 是否为 labels.Nothing()，也就是说如果你在写代码时通过 List 并传递 labels.Nothing() 的话，其最终还是会遍历一遍所有对象，只是并不会返回任何内容，会有这么一个无效遍历的过程。
你可能会说，我为什么要传递 labels.Nothing() 给 List 方法呢，这不是纯有毛病吗？是的，在你明确知道要传递给 List 什么时，你是不会传递 labels.Nothing() 的，但问题就在于传递的是个 Selector 的 interface，而最终传递的对象往往是通过其他函数获得的，类似

``` go
selector, _ := metav1.LabelSelectorAsSelector(labelSelector)
objs := objLister.List(selector)
```

类似这种写法，有谁会注意这里的 selector 到底是什么呢，可以搜一下 k8s 代码，很多地方有这个写法，而且传入的 labelSelector 经常就是某个结构里面定义的 *labels.Selector 的属性，默认就是 nil，而上面的函数在 labelSelector 是 nil 时返回的 selector 就是 labels.Nothing()，然后 List 的时候就会来一遍无效遍历，看起来好像很忙，但是是没有任何产出的瞎忙。这个问题也已经提了 PR [optimize ListAll and ListAllByNamespace to return directly when nothing to select](https://github.com/kubernetes/kubernetes/pull/132255/)，当前还在讨论是通过在 Selector 中添加新的方法，还是在此文件中添加 helper 函数来判断传入的 labelSelector 是否是 labels.Nothing()，但无论如何，这个问题是需要解决的。

# 总结

其实都是一些平时不会注意的地方，通过 pprof 或者仔细看代码的话，还是可以轻松的发现的，修复方式也比较简单。此外也发现一些其他的会影响 InterPodAffinity 性能的小问题，就不在这里说了，如果再提 PR 的话，会单独再开一篇来描述。

