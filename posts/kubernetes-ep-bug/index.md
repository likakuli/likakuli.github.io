# Kubernetes惊天地泣鬼神之大bug


最近docker one的交流群里发出了一篇文章，[Kubernetes 惊天地泣鬼神之大Bug](https://zhuanlan.zhihu.com/p/37217575) ，估计很多人看完文章的反应和我一样，心中万马奔腾，自己的集群会不会也有这个问题 ？？？

![](/media/funny/3c77c63b18cafd0f1d0706f332b88574_hd.jpg)

文中对于bug的影响和产生的原因已经描述的很清楚了，但看完之后我有一点疑问，文中所说的复现条件(

`陆续创建、删除、创建 Kubernetes service 对象，然后"kubectl delete svc xxx"删掉创建时间靠前的 service，也就是往 service event list 末尾插入了一条 resourceVersion 比较小的记录，这将使得 controller-manager 去从已经爬过的 service event list 位置重新爬取重放，然后就重放了 service 的 ADDED、DELETED event，于是 controller-manager 内存里缓存的 service 对象被删除，导致 EndpointController 删除了“不存在的”service 对应的 endpoints。`)在日常操作中其实会经常出现，那岂不是很多集群都会存在这个问题，而且这个bug影响这么严重，为什么现在才被报出来？基于上述疑问，本文主要是到源码中一探究竟，源码版本1.9.2，按照文中指引可以快速定位到问题代码所在位置，话不多说，直接上代码，如下：

```go
// watchHandler watches w and keeps *resourceVersion up to date.
func (r *Reflector) watchHandler(w watch.Interface, resourceVersion *string, errc chan error, stopCh <-chan struct{}) error {
	start := r.clock.Now()
	eventCount := 0

	// Stopping the watcher should be idempotent and if we return from this function there's no way
	// we're coming back in with the same watch interface.
	defer w.Stop()
	// update metrics
	defer func() {
		r.metrics.numberOfItemsInWatch.Observe(float64(eventCount))
		r.metrics.watchDuration.Observe(time.Since(start).Seconds())
	}()

loop:
	for {
		select {
		case <-stopCh:
			return errorStopRequested
		case err := <-errc:
			return err
		case event, ok := <-w.ResultChan():
			if !ok {
				break loop
			}
			if event.Type == watch.Error {
				return apierrs.FromObject(event.Object)
			}
			if e, a := r.expectedType, reflect.TypeOf(event.Object); e != nil && e != a {
				utilruntime.HandleError(fmt.Errorf("%s: expected type %v, but watch event object had type %v", r.name, e, a))
				continue
			}
			meta, err := meta.Accessor(event.Object)
			if err != nil {
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
				continue
			}
			newResourceVersion := meta.GetResourceVersion()
			switch event.Type {
			case watch.Added:
				err := r.store.Add(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to add watch event object (%#v) to store: %v", r.name, event.Object, err))
				}
			case watch.Modified:
				err := r.store.Update(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to update watch event object (%#v) to store: %v", r.name, event.Object, err))
				}
			case watch.Deleted:
				// TODO: Will any consumers need access to the "last known
				// state", which is passed in event.Object? If so, may need
				// to change this.
				err := r.store.Delete(event.Object)
				if err != nil {
					utilruntime.HandleError(fmt.Errorf("%s: unable to delete watch event object (%#v) from store: %v", r.name, event.Object, err))
				}
			default:
				utilruntime.HandleError(fmt.Errorf("%s: unable to understand watch event %#v", r.name, event))
			}
			*resourceVersion = newResourceVersion
			r.setLastSyncResourceVersion(newResourceVersion)
			eventCount++
		}
	}

	watchDuration := r.clock.Now().Sub(start)
	if watchDuration < 1*time.Second && eventCount == 0 {
		r.metrics.numberOfShortWatches.Inc()
		return fmt.Errorf("very short watch: %s: Unexpected watch close - watch lasted less than a second and no items received", r.name)
	}
	glog.V(4).Infof("%s: Watch close - %v total %v items received", r.name, r.expectedType, eventCount)
	return nil
}
```

这是文中指出的具体BUG所在，用错误的newResourceVersion去给resourceVersion和lastSyncResourceVersion赋了值，所以接下来就是要找这两个属性在什么地方被用到了，经过逐步查找代码引用，最后确定lastSyncResourceVersion和这个bug无关，那问题就出在了resourceVersion上，可以看到上面的方法中resourceVersion是通过指针传进来的，也就是说会影响到外面使用到这个属性的地方，继续查看watchHandler被调用的代码，如下：

```go
// ListAndWatch first lists all items and get the resource version at the moment of call,
// and then use the resource version to watch.
// It returns error if ListAndWatch didn't even try to initialize watch.
func (r *Reflector) ListAndWatch(stopCh <-chan struct{}) error {
	glog.V(3).Infof("Listing and watching %v from %s", r.expectedType, r.name)
	var resourceVersion string

	// Explicitly set "0" as resource version - it's fine for the List()
	// to be served from cache and potentially be delayed relative to
	// etcd contents. Reflector framework will catch up via Watch() eventually.
	options := metav1.ListOptions{ResourceVersion: "0"}
	r.metrics.numberOfLists.Inc()
	start := r.clock.Now()
	list, err := r.listerWatcher.List(options)
	if err != nil {
		return fmt.Errorf("%s: Failed to list %v: %v", r.name, r.expectedType, err)
	}
	r.metrics.listDuration.Observe(time.Since(start).Seconds())
	listMetaInterface, err := meta.ListAccessor(list)
	if err != nil {
		return fmt.Errorf("%s: Unable to understand list result %#v: %v", r.name, list, err)
	}
	resourceVersion = listMetaInterface.GetResourceVersion()
	items, err := meta.ExtractList(list)
	if err != nil {
		return fmt.Errorf("%s: Unable to understand list result %#v (%v)", r.name, list, err)
	}
	r.metrics.numberOfItemsInList.Observe(float64(len(items)))
	if err := r.syncWith(items, resourceVersion); err != nil {
		return fmt.Errorf("%s: Unable to sync list result: %v", r.name, err)
	}
	r.setLastSyncResourceVersion(resourceVersion)

	resyncerrc := make(chan error, 1)
	cancelCh := make(chan struct{})
	defer close(cancelCh)
	go func() {
		resyncCh, cleanup := r.resyncChan()
		defer func() {
			cleanup() // Call the last one written into cleanup
		}()
		for {
			select {
			case <-resyncCh:
			case <-stopCh:
				return
			case <-cancelCh:
				return
			}
			if r.ShouldResync == nil || r.ShouldResync() {
				glog.V(4).Infof("%s: forcing resync", r.name)
				if err := r.store.Resync(); err != nil {
					resyncerrc <- err
					return
				}
			}
			cleanup()
			resyncCh, cleanup = r.resyncChan()
		}
	}()

	for {
		// give the stopCh a chance to stop the loop, even in case of continue statements further down on errors
		select {
		case <-stopCh:
			return nil
		default:
		}

		timeoutSeconds := int64(minWatchTimeout.Seconds() * (rand.Float64() + 1.0))
		options = metav1.ListOptions{
			ResourceVersion: resourceVersion,
			// We want to avoid situations of hanging watchers. Stop any wachers that do not
			// receive any events within the timeout window.
			TimeoutSeconds: &timeoutSeconds,
		}

		r.metrics.numberOfWatches.Inc()
		w, err := r.listerWatcher.Watch(options)
		if err != nil {
			switch err {
			case io.EOF:
				// watch closed normally
			case io.ErrUnexpectedEOF:
				glog.V(1).Infof("%s: Watch for %v closed with unexpected EOF: %v", r.name, r.expectedType, err)
			default:
				utilruntime.HandleError(fmt.Errorf("%s: Failed to watch %v: %v", r.name, r.expectedType, err))
			}
			// If this is "connection refused" error, it means that most likely apiserver is not responsive.
			// It doesn't make sense to re-list all objects because most likely we will be able to restart
			// watch where we ended.
			// If that's the case wait and resend watch request.
			if urlError, ok := err.(*url.Error); ok {
				if opError, ok := urlError.Err.(*net.OpError); ok {
					if errno, ok := opError.Err.(syscall.Errno); ok && errno == syscall.ECONNREFUSED {
						time.Sleep(time.Second)
						continue
					}
				}
			}
			return nil
		}

		if err := r.watchHandler(w, &resourceVersion, resyncerrc, stopCh); err != nil {
			if err != errorStopRequested {
				glog.Warningf("%s: watch of %v ended with: %v", r.name, r.expectedType, err)
			}
			return nil
		}
	}
}
```

可以看到watchHandler是在for循环中被调用的，这个是用来保证不间断的watch，前一次的Watch和watchHandler调用中传入了resourceVersion，在watchHandler执行过程中被修改为了其他值，watchHandler调用退出之后进行下次调用时，resourceVersion又作为下一次Watch方法调用的参数被传递，这时候如果resourceVersion被改为旧值就会出现这个BUG，这也印证了之前说的在每次endpoint被删除时都会有一条watch api被调用的日志。

发现了问题所在，回过头来再看下上述两段代码，watchHandler里定义了对接收到的数据的处理逻辑，在一个for循环中进行，ListAndWatch的Watch和watchHandler的调用也在for循环中调用，只有在watchHandler返回，并且返回的err是nil的情况下才会继续watch，才有可能触发这个bug，只要watchHandler不退出或退出且返回err不为空，即使中间出现过resourceVersion被错误赋值的情况也不会导致bug的出现，这也就解释了我的疑问，为什么这个bug影响这么严重，现在才被注意到，因为这个bug不会经常出现。在我们用到的1.6.7的集群中执行kubectl get ep --all-namespaces得到的结果中除了确实是刚创建的svc外的endpoint外，其余endpoint的存活时间都已有几十到几百天，也就是在相当长一段时间内并没有出现这个bug，虽然出现的话影响很严重。知乎的评论中有人说按照复现方式进行操作但是没有出现这个bug也是这个道理。

综上，bug复现需要满足两点：

1. watchHandler正常退出，对于1.9.2版本来说就是event, ok := <-w.ResultChan()，ok为false，也就是channel被关闭或者返回为空。
2. 退出前接收到的最后一个Event的type为delete，而且是之前创建的，也就是resourceVersion是旧的。

这个bug影响确实严重，但是不会经常出现，像bug的发现者说的那样频繁出现的情况，可能还需要继续排查下为什么watchHander的调用会频繁的正常退出，可能这才是问题真正所在。



更多精彩内容可关注微信订阅号：幼儿园小班工程师
