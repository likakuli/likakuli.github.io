# Kube-apiserver watch实现


List-Watch是kubernetes的核心机制。组件kubelet、kube-controller-manager、kube-scheduler需要监控各种资源(pod、service等)的变化，当这些对象发生变化时(add、delete、update)，kube-apiserver会主动通知这些组件。这个过程类似一个发布-订阅系统。本文章将从代码角度探究一下list-watch的实现方式。

> 转载自https://zhuanlan.zhihu.com/p/33335726，有修改

## **第一部分：**kube-apiserver对etcd的List-watch机制

**流程示意图**

![img](https://pic2.zhimg.com/80/v2-a440358b42457f4bfffe95b0ddff2fe9_hd.jpg)

**构建PodStorage**

1. kube-apiserver针对每一类资源(pod、service、endpoint、replication controller、depolyments)都会构建Storage对象，如:PodStorage；
2. PodStorage.Pod.Store封装了对etcd的操作；
3. store.CompleteWithOptions会调用etcdOptions.GetRESTOptions，此方法将

**调用generic.UndecoratedStorage创建无cache的Store；**

**或者调用genericregistry.StorageWithCacher创建带Cache的Store；**

![img](https://pic4.zhimg.com/80/v2-5682d6d1a894c81f8c03a51651a6082f_hd.jpg)

**StorageWithCacher**

1. 调用NewCacherFromConfig，将创建Cacher对象；

![img](https://pic4.zhimg.com/80/v2-692a1b630fe492fe9bd51ec951ab08d7_hd.jpg)

**创建Cacher**

1. 首先，创建watchCache对象和cacheListerWatcher对象，cacheListWatcher对象是ListerWatcher接口实现，实现了List()和Watch()方法；

2. 构建Cacher对象，主要的数据成员：watchCache、reflector、watchers及incoming channel；

3. 1. watchCache是一个cache，用来存储apiserver从etcd那里watch到的对象；
   2. watchers是一个map，map的值类型为cacheWatcher，当kubelet、kube-scheduler需要watch某类资源时，他们会向kube-apiserver发起watch请求，kube-apiserver就会生成一个cacheWatcher，cacheWatcher负责将watch的资源通过http从apiserver传递到kubelet、kube-scheduler；
   3. Reflector对象，主要数据成员：ListerWatcher，ListerWatcher是接口对象，包括方法List()和Watch()；listerWatcher包装了Storage，主要是将watch到的对象存到watchCache中；
   4. incoming channel接收watchCacheEvent；

4. 协程调用cacher.dispatchEvents，watchCache将incoming channel接收watchCacheEvent添加到watchers的inputChan中；

5. 协程调用cacher.startCaching;

![img](https://pic4.zhimg.com/80/v2-fb0098596842640ba373888916e48a03_hd.jpg)

**StartCaching**

1. 执行cacheListerWatcher的List方法和Watch方法；
2. 调用reflector的watchHandler方法；

**cacheListWatcher.List／cacheListWatcher.Watch**

1. List方法将调用storage.List方法，这里是etcdHelper.List方法；
2. Watch方法将调用storage.watch方法，这里是etcdHelper.WatchList方法；

![img](https://pic3.zhimg.com/80/v2-7d72d3254188269f09a780e7c2535b5e_hd.jpg)

**etcdHelper.List／etcdHelper.Watch**

1. etcdHelper对象是Storage接口对象的实现;

2. etcdHelper的List方法：

3. 1. 获取etcd的对象（包括resourceVersion信息）；

4. etcdHelper的WatchList方法：

5. 1. 创建etcdWatcher；
   2. etcdWatcher对象，实现了Watch接口；
   3. etcdWatcher对象，主要的数据成员是etcdIncoming channel和outgoing channel；
   4. 协程执行etcdWatcher.translate；
   5. 最后，协程运行etcdWatcher.etcdWatch；

![img](https://pic1.zhimg.com/80/v2-8d8da3a1094d1d8b9e492ce7b2ef2aa4_hd.jpg)

**etcdWatcher.etcdWatch**

1. 如果resourceVersion==0, 运行etcdGetInitialWatchState(),获取所有的pods，并将结果输入到etcdIncoming channel;
2. 之后，不停的调用watcher.Next()，并将结果输入到etcdIncoming channel;

![img](https://pic4.zhimg.com/80/v2-f38da6d32e1de064d2d07072d124a453_hd.jpg)

**etcdWatcher.translate**

1. 读取etcdIncoming channel信息；
2. 调用etcdWatcher.sendResult进行转化;
3. 发送到outgoing channel；

![img](https://pic3.zhimg.com/80/v2-189b4e61b547ba220ca6c85ee3af115e_hd.jpg)

**reflector.watchHandler**

1. 读取outgoing channel信息，操作watchCache；

![img](https://pic3.zhimg.com/80/v2-a060020e7631a45a5d1cd676bab4c9fa_hd.jpg)

**操作watchCache**

![img](https://pic4.zhimg.com/80/v2-c75fcd3e4cd1b755d283b0f8cde5740b_hd.jpg)

**处理事件watchCache.processEvent**

1. 创建watchCacheEvent
2. 调用watchCache.updateCache，更新watchCache;

![img](https://pic1.zhimg.com/80/v2-947dc83646b5b4135f95c3265577edbc_hd.jpg)

到此分析完kube-apiserver对etcd的watch机制，除此之外，kube-apiserver会向其他组件提供watch接口，下面将分析kube-apiserver的watch API。

## **第二部分：kube-apiserver的watch restful API**

kube-apiserver提供watch restful API给其他组件(kubelet、kube-controller-manager、kube-scheduler、kube-proxy)。watch restful API的处理流程和PUT、DELETE、GET等REST API处理流程类似。

**流程示意图**

![img](https://pic4.zhimg.com/80/v2-9f7daa341fd0b919f295175ea90b3acb_hd.jpg)

**registerResourceHandlers**

![img](https://pic4.zhimg.com/80/v2-1ef10ae851dd585ef6c17e801b19b7e3_hd.jpg)

**ListResource**

1.调用rw.watch方法，这里将会调用Store.watch；

2.调用serveWatch方法；

![img](https://pic2.zhimg.com/80/v2-75857fb8204b80f166becff887cb239d_hd.jpg)

**Store.watch**

1. 调用Storage.Watch方法和Storage.WatchList方法，这里将调用Cacher.watch方法和Cacher.WatchList方法

![img](https://pic2.zhimg.com/80/v2-1f89acacb17824b6c093bb95c6bbe825_hd.jpg)

**Cacher.watch**

1. watch方法中将调用newCacheWatcher；
2. newCacheWatcher方法：
3. 生成一个watcher，并将watcher插入到cacher.watchers中；
4. 协程调用cacheWatcher.process方法，此方法将会操作input channel的消息；

![img](https://pic4.zhimg.com/80/v2-8ab019a846bce30acf656a33cf658c03_hd.jpg)

**操作input channel**

1. 读取input channel的信息，并调用sendWatchCacheEvent方法；

![img](https://pic4.zhimg.com/80/v2-f724e294d4741bcc400d6e8606fb0abb_hd.jpg)

**sendWatchCacheEvent**

1. kube-apiserver的watch会带过滤功能；
2. 对watchCacheEvent进行Filter，发送到cacher.result channel中；

![img](https://pic3.zhimg.com/80/v2-6dfb5e5ceaaff90a62ee85aeb14dcd02_hd.jpg)


