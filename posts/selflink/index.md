# k8s 骨灰级玩家都不知道的属性 - SelfLink


> 本篇介绍一个可能连骨灰级 K8S 玩家都很少关注的属性 - **SelfLink**，以及这么一个名不见经传的小趴菜是怎么影响 kube-apiserver 性能的。大家可能没注意过他，但在分析 kube-apiserver 内存消耗时肯定见到过令人闻风丧胆的 DeepCopyObejct，本文会介绍他们之间的连续

# 背景

前面已经有了一系列有关 informer 和 kube-apiserver 的文章，定位和内容深度偏中上，如果没有基础的话可能不容易看懂。本篇介绍一个可能连骨灰级 K8S 玩家都很少关注的属性 - **SelfLink**，以及这么一个名不见经传的小趴菜是怎么影响 kube-apiserver 性能的。

# 作用

SelfLink 到底干什么的呢？

> `SelfLink` is a URL representing a given object. It is part of `ObjectMeta` and `ListMeta` which means that it is part of every single Kubernetes object.

SelfLink 从哪个版本引入的，存在的意义是什么，已经不得而知了，但可以明确的是从 v1.24 开始被正式废弃了。那为什么还要提一下他呢，因为他虽已不在江湖，但江湖上仍流传着他的传说。这个属性不仅没什么作用，甚至还有一些副作用，尤其你如果使用的是 v1.24 前的版本，更要注意下。

# 被废弃

被废弃的原因有两点：

- 在 generic-apiserver 中以非常特殊的方式处理 - 它是在序列化对象之前设置的唯一字段（因为这是设置它所需的所有必要信息的唯一位置）；
- 具有不可忽略性能影响 - 构造该值需要执行几次内存分配；

# 原理

## DeepCopyObject

上一篇 [kube-apiserver 内存优化进阶](https://mp.weixin.qq.com/s/RrBOHRSztsnkp5M2_32VrQ) 中介绍了从序列化的角度去优化 kube-apiserver 的内存使用，还不懂的建议先看前一篇。SelfLink 的处理就在序列化之前。

```go
// ServeHTTP serves a series of encoded events via HTTP with Transfer-Encoding: chunked
// or over a websocket connection.
func (s *WatchServer) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	...

	for {
		select {
		case <-done:
			return
		case <-timeoutCh:
			return
		case event, ok := <-ch:
			...

			obj := s.Fixup(event.Object)
			if err := s.EmbeddedEncoder.Encode(obj, buf); err != nil {
				// unexpected error
				utilruntime.HandleError(fmt.Errorf("unable to encode watch object %T: %v", obj, err))
				return
			}

			...
		}
	}
}

// TODO: the functionality in this method and in WatchServer.Serve is not cleanly decoupled.
func serveWatch(watcher watch.Interface, scope *RequestScope, mediaTypeOptions negotiation.MediaTypeOptions, req *http.Request, w http.ResponseWriter, timeout time.Duration) {
	...
  
	server := &WatchServer{
		...

		Fixup: func(obj runtime.Object) runtime.Object {
			result, err := transformObject(ctx, obj, options, mediaTypeOptions, scope, req)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("failed to transform object %v: %v", reflect.TypeOf(obj), err))
				return obj
			}
			// When we are transformed to a table, use the table options as the state for whether we
			// should print headers - on watch, we only want to print table headers on the first object
			// and omit them on subsequent events.
			if tableOptions, ok := options.(*metav1.TableOptions); ok {
				tableOptions.NoHeaders = true
			}
			return result
		},

		TimeoutFactory: &realTimeoutFactory{timeout},
	}

	server.ServeHTTP(w, req)
}

// transformObject takes the object as returned by storage and ensures it is in
// the client's desired form, as well as ensuring any API level fields like self-link
// are properly set.
func transformObject(ctx context.Context, obj runtime.Object, opts interface{}, mediaType negotiation.MediaTypeOptions, scope *RequestScope, req *http.Request) (runtime.Object, error) {
	if co, ok := obj.(runtime.CacheableObject); ok {
		if mediaType.Convert != nil {
			// Non-nil mediaType.Convert means that some conversion of the object
			// has to happen. Currently conversion may potentially modify the
			// object or assume something about it (e.g. asTable operates on
			// reflection, which won't work for any wrapper).
			// To ensure it will work correctly, let's operate on base objects
			// and not cache it for now.
			//
			// TODO: Long-term, transformObject should be changed so that it
			// implements runtime.Encoder interface.
			return doTransformObject(ctx, co.GetObject(), opts, mediaType, scope, req)
		}
	}
	return doTransformObject(ctx, obj, opts, mediaType, scope, req)
}

func doTransformObject(ctx context.Context, obj runtime.Object, opts interface{}, mediaType negotiation.MediaTypeOptions, scope *RequestScope, req *http.Request) (runtime.Object, error) {
	...
  
	if err := setObjectSelfLink(ctx, obj, req, scope.Namer); err != nil {
		return nil, err
	}

	...
}
```

总结下就是 WatchServer 中包含一个 FixUp 方法，用来在序列化最终要返回的 event 之前做一些操作，其中就包括为 event.object 设置 SelfLink，这是一个非标操作，整个过程只有他在序列化前做了特殊处理。

但为什么设置 SelfLink 会对性能有影响呢，不就是一个简单的赋值操作吗？这就涉及到上一篇介绍过的 CacheableObject 对象了，对于非 CacheableObject 没有影响。上一篇中我们提到了一个神奇的地方

> cacheWatcher 的 input chan 的 event 对象的 object 有可能是正常的资源对象，例如 Pod，也有可能是 CacheableObject 对象，而真正的资源对象则保存在 CacheableObject 的 object 中

所以上面代码 transformObject 在处理时也判断了类型，我们只关注 CacheableObject 这部分 

`doTransformObject(ctx, co.GetObject(), opts, mediaType, scope, req)` 其中调用了 co.GetObject() 从 CacheableObject 中获取真实的资源对象（如 Pod），而这个方法会对资源对象执行一个 deepcopy 操作

```go
// GetObject implements runtime.CacheableObject interface.
// It returns deep-copy of the wrapped object to return ownership of it
// to the called according to the contract of the interface.
func (o *cachingObject) GetObject() runtime.Object {
	o.lock.RLock()
	defer o.lock.RUnlock()
	return o.object.DeepCopyObject().(metaRuntimeInterface)
}
```

针对 v1.24 之前的版本，在创建 cachingObject 对象时就已经对资源对象做了一次 deepcopy 操作了

```go
// newCachingObject performs a deep copy of the given object and wraps it
// into a cachingObject.
// An error is returned if it's not possible to cast the object to
// metav1.Object type.
func newCachingObject(object runtime.Object) (*cachingObject, error) {
	if obj, ok := object.(metaRuntimeInterface); ok {
		result := &cachingObject{object: obj.DeepCopyObject().(metaRuntimeInterface)}
		result.serializations.Store(make(serializationsCache))
		return result, nil
	}
	return nil, fmt.Errorf("can't cast object to metav1.Object: %#v", object)
}
```

在 doTransformObject 时又进行了一次 deepcopy，这不仅需要分配内存，也会额外占用内存空间，对性能和内存使用量都会负面影响。

而针对 v1.24 及之后的版本，cachingObject 做了优化，支持了 lazy deepcopy 的能力，即在创建 cachingObject 对象时，并不会无脑 deepcopy，而是在真正的去修改其 object 属性时再去执行 deepcopy

```go
// newCachingObject performs a deep copy of the given object and wraps it
// into a cachingObject.
// An error is returned if it's not possible to cast the object to
// metav1.Object type.
func newCachingObject(object runtime.Object) (*cachingObject, error) {
	if obj, ok := object.(metaRuntimeInterface); ok {
		result := &cachingObject{
			object:     obj,
			deepCopied: false,
		}
		result.serializations.Store(make(serializationsCache))
		return result, nil
	}
	return nil, fmt.Errorf("can't cast object to metav1.Object: %#v", object)
}

func (o *cachingObject) SetSelfLink(selfLink string) {
	o.conditionalSet(
		func() bool { return o.object.GetSelfLink() == selfLink },
		func() { o.object.SetSelfLink(selfLink) },
	)
}

func (o *cachingObject) conditionalSet(isNoop func() bool, set func()) {
	if fastPath := func() bool {
		o.lock.RLock()
		defer o.lock.RUnlock()
		return isNoop()
	}(); fastPath {
		return
	}
	o.lock.Lock()
	defer o.lock.Unlock()
	if isNoop() {
		return
	}
	if !o.deepCopied {
		o.object = o.object.DeepCopyObject().(metaRuntimeInterface)
		o.deepCopied = true
	}
	o.invalidateCacheLocked()
	set()
}
```

deepcopy 执行条件：**object 对象被修改了，且之前没有进行过 deepcopy 操作**。

最后可以看到 v1.24 相对之前的版本，少了"**两次**"资源对象的 deepcopy 操作。其中一次是 SelfLink 造成的，一次是在 newCachingObject 的时候发生的。

## 缘起

严格的说，上面刚提到的"两次"其实并不严谨，更准确的说是一到两次。这里就又涉及到了一个细节：

```go
func setCachingObjects(event *watchCacheEvent, versioner storage.Versioner) {
	switch event.Type {
	case watch.Added, watch.Modified:
		if object, err := newCachingObject(event.Object); err == nil {
			event.Object = object
		} else {
			klog.Errorf("couldn't create cachingObject from: %#v", event.Object)
		}
		// Don't wrap PrevObject for update event (for create events it is nil).
		// We only encode those to deliver DELETE watch events, so if
		// event.Object is not nil it can be used only for watchers for which
		// selector was satisfied for its previous version and is no longer
		// satisfied for the current version.
		// This is rare enough that it doesn't justify making deep-copy of the
		// object (done by newCachingObject) every time.
	case watch.Deleted:
		// Don't wrap Object for delete events - these are not to deliver any
		// events. Only wrap PrevObject.
		if object, err := newCachingObject(event.PrevObject); err == nil {
			// Update resource version of the object.
			// event.PrevObject is used to deliver DELETE watch events and
			// for them, we set resourceVersion to <current> instead of
			// the resourceVersion of the last modification of the object.
			updateResourceVersion(object, versioner, event.ResourceVersion)
			event.PrevObject = object
		} else {
			klog.Errorf("couldn't create cachingObject from: %#v", event.Object)
		}
	}
}
```

Cacher 消费其 incoming chan，在去给每个筛选后的 cacheWatcher 发送 event 时，会执行 setCachingObjects 将资源对象做个封装，封装成 cachingObject，并将 cachingObject 设置为 event 的 Object 或者 PrevObject，具体设置到哪个属性上面，和 event type 有关，可以看上述代码，对于非 Deleted 类型，设置的是 event.Object，而对于 Deleted 类型的事件来说，设置的是 PrevObject，因为 Deleted 事件本身并不具备有效负载。这又和之前提的相呼应

> cacheWatcher 的 input chan 的 event 对象的 object 有可能是正常的资源对象，例如 Pod，也有可能是 CacheableObject 对象，而真正的资源对象则保存在 CacheableObject 的 object 中

继续看上面的代码，在处理 Deleted 类型时，还做了一个操作 updateResourceVersion，其作用是将 event 的 resourceVersion 设置给 object，为什么要做这个处理呢？不知道大家还记不记得之前那篇 [kubernetes 月光宝盒 - 时间倒流](https://mp.weixin.qq.com/s/B1OTBSIY7I-4TF0LaxtQUA) 的文章，这里正好对应了里面介绍的第二个时间回溯现象

> PrevObject 的 ResourceVersion 是一个过去时的值，例如连续创建三个 Pod，再删除第一个 Pod，则此时返回的 watchEvent 的 Object 对象的 ResourceVersion 是第一个 Pod 创建后的 ResourceVersion，但实际情况是最新的 ResourceVersion 已经随着后两个 Pod 的创建而递增了。也就是此时客户端 watch 到一个 delete event 之后，客户端 informer 所使用的 reflector 内部维护的 resourceVersion 已经不对了，是一个历史值，如果此时发生一个问题（例如网络闪断）需要去重新 watch 时，会使用这个错误的 resourceVersion，也就是这个版本之后的所有符合条件的事件会再次接收并处理一遍，真正的回到过去，历史重演。

在 v1.9 修复这个"惊天大 bug" 的时候，是在 cacheWatcher 消费自己 input chan 中的 event 发送给其 result chan 时处理的

```go
func (c *cacheWatcher) convertToWatchEvent(event *watchCacheEvent) *watch.Event {
	if event.Type == watch.Bookmark {
		return &watch.Event{Type: watch.Bookmark, Object: event.Object.DeepCopyObject()}
	}

	curObjPasses := event.Type != watch.Deleted && c.filter(event.Key, event.ObjLabels, event.ObjFields)
	oldObjPasses := false
	if event.PrevObject != nil {
		oldObjPasses = c.filter(event.Key, event.PrevObjLabels, event.PrevObjFields)
	}
	if !curObjPasses && !oldObjPasses {
		// Watcher is not interested in that object.
		return nil
	}

	switch {
	case curObjPasses && !oldObjPasses:
		return &watch.Event{Type: watch.Added, Object: getMutableObject(event.Object)}
	case curObjPasses && oldObjPasses:
		return &watch.Event{Type: watch.Modified, Object: getMutableObject(event.Object)}
	case !curObjPasses && oldObjPasses:
		// return a delete event with the previous object content, but with the event's resource version
		oldObj := getMutableObject(event.PrevObject)
		// We know that if oldObj is cachingObject (which can only be set via
		// setCachingObjects), its resourceVersion is already set correctly and
		// we don't need to update it. However, since cachingObject efficiently
		// handles noop updates, we avoid this microoptimization here.
		updateResourceVersion(oldObj, c.versioner, event.ResourceVersion)
		return &watch.Event{Type: watch.Deleted, Object: oldObj}
	}

	return nil
}
```

可以看到在最后一个 case 分支的处理逻辑中，仍然存在一个 updateResourceVersion 的操作，仍然是使用了 event.ResourceVersion 去覆盖了 oldObject 也即是上面提的 PrevObject 的 ResourceVersion 属性。在 v1.9 中其实只有后面这一次的调用，而在 v1.24 中存在着两次调用，上面代码中也特别注释了如果 oldObject 是 cachingObject 的话，他的 resourceversion 已经被正确的设置了，我们不需要再次设置它。但是由于 cachingObject 可以高效的处理 noop 更新操作，所以这里忽略了这个微乎其微的优化。cachingObject 为什么可以高效的处理 noop 是在其 conditionalSet 方法中实现的，他会判断要设置的值和当前的值是否一致，是的话就代表不需要执行更新操作，直接返回即可。

## 到底几次

所以到底是少了几次 deepcopy 呢？

这要根据 event type 来看，对于非 Deleted 类型的 event 来说，由于他们并不需要执行 updateResourceVersion 的操作，也就不会触发 deepcopy，而 SelfLink 是需要两次 deepcopy 的，所以是多了两次 deepcopy，而针对 Deleted 类型来说，由于其本身就需要一次 deepcopy（虽然两次 updateResourceVersion，但第二次属于 noop，直接返回），所以只是多了一次 deepcopy。

综上，对于 Deleted 类型的 event，去掉 SelfLink 可以减少一次资源对象的 deepcopy，而针对其他类型的 event，去掉 SelfLink 则可以减少两次 deepcopy。

# 总结

通过分析 SelfLink，我们又重新复习了 Cacher、cacheWatcher 甚至远古时期的惊天大"bug"，分析了到底他对性能有什么影响，去掉他之后优化了多少。

不知道你有没有发现，过去这么多篇的文章并不是独立的，他们之间总是存在着千丝万缕的联系，而这些关系通过各种细节串联起来，点到线，线到面，最后成为一个网，或者一个图，成为知识体系。他不仅具备空间属性，还具备时间属性。如果你不知道之前版本的问题或者实现的话，单看当前的实现可能不容易体会到他的巧妙之处，或者印象并不深刻。

从问题中来，沿着时间轴把相关的线索都梳理一遍，享受把这些蛛丝马迹串起来的过程，享受知识体系化的过程，体会并享受大佬们解决问题的思路。

谁说 K8S 越来越复杂？我看挺有意思的嘛，而且移除 SelfLink 不也是在做减法。

最后，针对上述描述如有问题，或者想交流的话，可以添加笔者 VX：YlikakuY，班门弄斧，欢迎互喷~

