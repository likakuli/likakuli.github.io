# docker hang问题排查


> 转载自组内同事

### 1. 背景

最近升级了一版kubelet，修复因kubelet删除Pod慢导致平台删除集群超时的问题。在灰度redis隔离集群的时候，发现升级kubelet并重启服务后，少量宿主状态变成了NotReady，并且回滚kubelet至之前版本，宿主状态仍然是NotReady。查看宿主状态时提示 'container runtime is down' ，根据经验，此时一般就是容器运行时出了问题。弹性云使用的容器运行时是docker，我们就去检查docker的状态，检测结果如下：

- docker ps 查看所有容器状态，执行正常
- docker inspect 查看某一容器详细状态，执行阻塞

典型的docker hang死行为。因为我们最近在升级docker版本，存量宿主docker的版本为1.13.1，并且在逐步升级至18.06.3，新宿主的docker版本都是18.06.3。docker hang死问题在1.13.1版本上表现得更彻底，在执行docker ps的时候就已经hang死了，一旦某个容器出了问题，docker就处于无响应状态；而docker 18.06.3做了一点小小的优化，在执行docker ps时去掉了针对容器级别的加锁操作，但是docker inspect依然会加容器锁，因此某一个容器出现问题，并不会造成docker服务不可响应，受影响的也仅仅是该容器，无法执行任何操作。

至于为什么以docker ps与docker inspect为指标检查docker状态，因为kubelet就是依赖这两个docker API获取容器状态。

所以，现在问题有二：

1. docker hang死的根因是什么？
2. docker hang死时，为什么重启kubelet，会导致宿主状态变为NotReady？

### 2.重启kubelet变更宿主状态

kubelet重启后宿主状态从Ready变为NotReady，这个问题相较docker hang死而言，没有那么复杂，所以我们先排查这个问题。

kubelet针对宿主会设置多个Condition，表明宿主当前所处的状态，比如宿主内存是否告急、线程数是否告急，以及宿主是否就绪。其中ReadyCondition表明宿主是否就绪，kubectl查看宿主状态时，展示的Statue信息就是ReadCondition的内容，常见的状态及其含义定义如下：

- Ready状态：表明当前宿主状态一切OK，能正常响应Pod事件
- NotReady状态：表明宿主的kubelet仍在运行，但是此时已经无法处理Pod事件。NotReady绝大多数情况都是容器运行时出了问题
- Unknown状态：表明宿主kubelet已停止运行

kubelet定义的ReadyCondition的判定条件如下：

```go
// defaultNodeStatusFuncs is a factory that generates the default set of
// setNodeStatus funcs
func (kl *Kubelet) defaultNodeStatusFuncs() []func(*v1.Node) error {
   ......
   setters = append(setters,
      nodestatus.OutOfDiskCondition(kl.clock.Now, kl.recordNodeStatusEvent),
      nodestatus.MemoryPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderMemoryPressure, kl.recordNodeStatusEvent),
      nodestatus.DiskPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderDiskPressure, kl.recordNodeStatusEvent),
      nodestatus.PIDPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderPIDPressure, kl.recordNodeStatusEvent),
      nodestatus.ReadyCondition(kl.clock.Now, kl.runtimeState.runtimeErrors, kl.runtimeState.networkErrors, validateHostFunc, kl.containerManager.Status, kl.recordNodeStatusEvent),
      nodestatus.VolumesInUse(kl.volumeManager.ReconcilerStatesHasBeenSynced, kl.volumeManager.GetVolumesInUse),
      // TODO(mtaufen): I decided not to move this setter for now, since all it does is send an event
      // and record state back to the Kubelet runtime object. In the future, I'd like to isolate
      // these side-effects by decoupling the decisions to send events and partial status recording
      // from the Node setters.
      kl.recordNodeSchedulableEvent,
   )
   return setters
}
```

深入nodestatus.ReadyCondition的实现可以发现，宿主是否Ready取决于很多条件，包含运行时判定、网络判定、基本资源判定等。这里我们只需关注运行时判定即可：

```go
func (s *runtimeState) runtimeErrors() []string {
   s.RLock()
   defer s.RUnlock()
   var ret []string
   if !s.lastBaseRuntimeSync.Add(s.baseRuntimeSyncThreshold).After(time.Now()) {  // 1
      ret = append(ret, "container runtime is down")
   }
   if s.internalError != nil {
      ret = append(ret, s.internalError.Error())
   }
   for _, hc := range s.healthChecks {                                            // 2
      if ok, err := hc.fn(); !ok {
         ret = append(ret, fmt.Sprintf("%s is not healthy: %v", hc.name, err))
      }
   }
   return ret
}
```

当出现如下两种状况之一时，则判定运行时检查不通过：

1. 距最近一次运行时同步操作的时间间隔超过指定阈值（默认30s）
2. 运行时健康检查未通过

那么，当时宿主的NotReady是由哪种状况引起的呢？结合kubelet日志分析，kubelet每隔5s就输出一条日志：

```shell
......
I0715 10:43:28.049240   16315 kubelet.go:1835] skipping pod synchronization - [container runtime is down]
I0715 10:43:33.049359   16315 kubelet.go:1835] skipping pod synchronization - [container runtime is down]
I0715 10:43:38.049492   16315 kubelet.go:1835] skipping pod synchronization - [container runtime is down]
......
```

因此，状况1是宿主NotReady的元凶。

我们继续分析为什么kubelet没有按照预期设置lastBaseRuntimeSync。kubelet启动时会创建一个goroutine，并在该goroutine中循环设置lastBaseRuntimeSync，循环如下：

```go
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
   ......
   go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)
   ......
}


func (kl *Kubelet) updateRuntimeUp() {
   kl.updateRuntimeMux.Lock()
   defer kl.updateRuntimeMux.Unlock()
   ......
   kl.oneTimeInitializer.Do(kl.initializeRuntimeDependentModules)
   kl.runtimeState.setRuntimeSync(kl.clock.Now())
}
```

正常情况下，kubelet每隔5s会将lastBaseRuntimeSync设置为当前时间，而宿主状态异常时，这个时间戳一直未被更新。也即updateRuntimeUp一直被阻塞在设置lastBaseRuntimeSync之前的某一步。我们只需逐个排查updateRuntimeUp内的函数调用即可，具体过程不再展示，最终的函数调用链路如下：

```shell
initializeRuntimeDependentModules -> kl.cadvisor.Start -> cc.Manager.Start -> self.createContainer -> m.createContainerLocked -> container.NewContainerHandler -> factory.CanHandleAndAccept -> self.client.ContainerInspect
```

由于某个容器状态异常，kubelet执行docker inspect操作也被hang死。

因此，重启kubelet引起宿主状态从Ready变为NotReady，其根因在于某个容器状态异常，执行docker inspect时被hang死。而如果docker inspect hang死发生在kubelet重启之后，则不会对宿主的Ready状态造成任何影响，因为oneTimeInitializer是sync.Once类型，也即仅仅会在kebelet启动时执行一次。那时kubelet仅仅是不能处理该Pod相关的任何事件，包含删除、变更等，但是仍然能够处理其他Pod的任意事件。

可能有人会问，为什么kubelet重启时访问docker inspect操作不加超时控制？确实，如果添加了超时控制，kubelet重启不会引起宿主状态变更。待详细挖掘后再来补充，我们先继续分析docker hang死的问题。

### 3. docker hang死

我们对docker hang死并不陌生，因为已经发生了好多起。其发生时的现象也多种多样。以往针对docker 1.13.1版本的排查都发现了一些线索，但是并没有定位到根因，最终绝大多数也是通过重启docker解决。而这一次发生在docker 18.06.3版本的docker hang死行为，经过我们4人小分队接近一周的望闻问切，终于确定了其病因。注意，docker hang死的原因不止一种，因此本处方并非是个万能药。

现在，我们掌握的知识仅仅是docker异常了，无法响应特定容器的docker inspect操作，而对详细信息则一无所知。

#### 3.1 链路跟踪

首先，我们希望对docker运行的全局状况有一个大致的了解，熟悉go语言开发的用户自然能联想到神器pprof。我们借助pprof描绘出了docker当时运行的蓝图：

```shell
goroutine profile: total 722373
717594 @ 0x7fe8bc202980 0x7fe8bc202a40 0x7fe8bc2135d8 0x7fe8bc2132ef 0x7fe8bc238c1a 0x7fe8bd56f7fe 0x7fe8bd56f6bd 0x7fe8bcea8719 0x7fe8bcea938b 0x7fe8bcb726ca 0x7fe8bcb72b01 0x7fe8bc71c26b 0x7fe8bcb85f4a 0x7fe8bc4b9896 0x7fe8bc72a438 0x7fe8bcb849e2 0x7fe8bc4bc67e 0x7fe8bc4b88a3 0x7fe8bc230711
#	0x7fe8bc2132ee	sync.runtime_SemacquireMutex+0x3e																/usr/local/go/src/runtime/sema.go:71
#	0x7fe8bc238c19	sync.(*Mutex).Lock+0x109																	/usr/local/go/src/sync/mutex.go:134
#	0x7fe8bd56f7fd	github.com/docker/docker/daemon.(*Daemon).ContainerInspectCurrent+0x8d												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/daemon/inspect.go:40
#	0x7fe8bd56f6bc	github.com/docker/docker/daemon.(*Daemon).ContainerInspect+0x11c												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/daemon/inspect.go:29
#	0x7fe8bcea8718	github.com/docker/docker/api/server/router/container.(*containerRouter).getContainersByName+0x118								/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/container/inspect.go:15
#	0x7fe8bcea938a	github.com/docker/docker/api/server/router/container.(*containerRouter).(github.com/docker/docker/api/server/router/container.getContainersByName)-fm+0x6a	/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/container/container.go:39
#	0x7fe8bcb726c9	github.com/docker/docker/api/server/middleware.ExperimentalMiddleware.WrapHandler.func1+0xd9									/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/middleware/experimental.go:26
#	0x7fe8bcb72b00	github.com/docker/docker/api/server/middleware.VersionMiddleware.WrapHandler.func1+0x400									/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/middleware/version.go:62
#	0x7fe8bc71c26a	github.com/docker/docker/pkg/authorization.(*Middleware).WrapHandler.func1+0x7aa										/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/pkg/authorization/middleware.go:59
#	0x7fe8bcb85f49	github.com/docker/docker/api/server.(*Server).makeHTTPHandler.func1+0x199											/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/server.go:141
#	0x7fe8bc4b9895	net/http.HandlerFunc.ServeHTTP+0x45																/usr/local/go/src/net/http/server.go:1947
#	0x7fe8bc72a437	github.com/docker/docker/vendor/github.com/gorilla/mux.(*Router).ServeHTTP+0x227										/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/github.com/gorilla/mux/mux.go:103
#	0x7fe8bcb849e1	github.com/docker/docker/api/server.(*routerSwapper).ServeHTTP+0x71												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router_swapper.go:29
#	0x7fe8bc4bc67d	net/http.serverHandler.ServeHTTP+0xbd																/usr/local/go/src/net/http/server.go:2694
#	0x7fe8bc4b88a2	net/http.(*conn).serve+0x652																	/usr/local/go/src/net/http/server.go:1830

4175 @ 0x7fe8bc202980 0x7fe8bc202a40 0x7fe8bc2135d8 0x7fe8bc2132ef 0x7fe8bc238c1a 0x7fe8bcc2eccf 0x7fe8bd597af4 0x7fe8bcea2456 0x7fe8bcea956b 0x7fe8bcb73dff 0x7fe8bcb726ca 0x7fe8bcb72b01 0x7fe8bc71c26b 0x7fe8bcb85f4a 0x7fe8bc4b9896 0x7fe8bc72a438 0x7fe8bcb849e2 0x7fe8bc4bc67e 0x7fe8bc4b88a3 0x7fe8bc230711
#	0x7fe8bc2132ee	sync.runtime_SemacquireMutex+0x3e																/usr/local/go/src/runtime/sema.go:71
#	0x7fe8bc238c19	sync.(*Mutex).Lock+0x109																	/usr/local/go/src/sync/mutex.go:134
#	0x7fe8bcc2ecce	github.com/docker/docker/container.(*State).IsRunning+0x2e													/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/container/state.go:240
#	0x7fe8bd597af3	github.com/docker/docker/daemon.(*Daemon).ContainerStats+0xb3													/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/daemon/stats.go:30
#	0x7fe8bcea2455	github.com/docker/docker/api/server/router/container.(*containerRouter).getContainersStats+0x1e5								/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/container/container_routes.go:115
#	0x7fe8bcea956a	github.com/docker/docker/api/server/router/container.(*containerRouter).(github.com/docker/docker/api/server/router/container.getContainersStats)-fm+0x6a	/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/container/container.go:42
#	0x7fe8bcb73dfe	github.com/docker/docker/api/server/router.cancellableHandler.func1+0xce											/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/local.go:92
#	0x7fe8bcb726c9	github.com/docker/docker/api/server/middleware.ExperimentalMiddleware.WrapHandler.func1+0xd9									/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/middleware/experimental.go:26
#	0x7fe8bcb72b00	github.com/docker/docker/api/server/middleware.VersionMiddleware.WrapHandler.func1+0x400									/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/middleware/version.go:62
#	0x7fe8bc71c26a	github.com/docker/docker/pkg/authorization.(*Middleware).WrapHandler.func1+0x7aa										/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/pkg/authorization/middleware.go:59
#	0x7fe8bcb85f49	github.com/docker/docker/api/server.(*Server).makeHTTPHandler.func1+0x199											/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/server.go:141
#	0x7fe8bc4b9895	net/http.HandlerFunc.ServeHTTP+0x45																/usr/local/go/src/net/http/server.go:1947
#	0x7fe8bc72a437	github.com/docker/docker/vendor/github.com/gorilla/mux.(*Router).ServeHTTP+0x227										/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/github.com/gorilla/mux/mux.go:103
#	0x7fe8bcb849e1	github.com/docker/docker/api/server.(*routerSwapper).ServeHTTP+0x71												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router_swapper.go:29
#	0x7fe8bc4bc67d	net/http.serverHandler.ServeHTTP+0xbd																/usr/local/go/src/net/http/server.go:2694
#	0x7fe8bc4b88a2	net/http.(*conn).serve+0x652																	/usr/local/go/src/net/http/server.go:1830

1 @ 0x7fe8bc202980 0x7fe8bc202a40 0x7fe8bc2135d8 0x7fe8bc2131fb 0x7fe8bc239a3b 0x7fe8bcbb679d 0x7fe8bcc26774 0x7fe8bd570b20 0x7fe8bd56f81c 0x7fe8bd56f6bd 0x7fe8bcea8719 0x7fe8bcea938b 0x7fe8bcb726ca 0x7fe8bcb72b01 0x7fe8bc71c26b 0x7fe8bcb85f4a 0x7fe8bc4b9896 0x7fe8bc72a438 0x7fe8bcb849e2 0x7fe8bc4bc67e 0x7fe8bc4b88a3 0x7fe8bc230711
#	0x7fe8bc2131fa	sync.runtime_Semacquire+0x3a																	/usr/local/go/src/runtime/sema.go:56
#	0x7fe8bc239a3a	sync.(*RWMutex).RLock+0x4a																	/usr/local/go/src/sync/rwmutex.go:50
#	0x7fe8bcbb679c	github.com/docker/docker/daemon/exec.(*Store).List+0x4c														/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/daemon/exec/exec.go:140
#	0x7fe8bcc26773	github.com/docker/docker/container.(*Container).GetExecIDs+0x33													/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/container/container.go:423
#	0x7fe8bd570b1f	github.com/docker/docker/daemon.(*Daemon).getInspectData+0x5cf													/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/daemon/inspect.go:178
#	0x7fe8bd56f81b	github.com/docker/docker/daemon.(*Daemon).ContainerInspectCurrent+0xab												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/daemon/inspect.go:42
#	0x7fe8bd56f6bc	github.com/docker/docker/daemon.(*Daemon).ContainerInspect+0x11c												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/daemon/inspect.go:29
#	0x7fe8bcea8718	github.com/docker/docker/api/server/router/container.(*containerRouter).getContainersByName+0x118								/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/container/inspect.go:15
#	0x7fe8bcea938a	github.com/docker/docker/api/server/router/container.(*containerRouter).(github.com/docker/docker/api/server/router/container.getContainersByName)-fm+0x6a	/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/container/container.go:39
#	0x7fe8bcb726c9	github.com/docker/docker/api/server/middleware.ExperimentalMiddleware.WrapHandler.func1+0xd9									/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/middleware/experimental.go:26
#	0x7fe8bcb72b00	github.com/docker/docker/api/server/middleware.VersionMiddleware.WrapHandler.func1+0x400									/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/middleware/version.go:62
#	0x7fe8bc71c26a	github.com/docker/docker/pkg/authorization.(*Middleware).WrapHandler.func1+0x7aa										/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/pkg/authorization/middleware.go:59
#	0x7fe8bcb85f49	github.com/docker/docker/api/server.(*Server).makeHTTPHandler.func1+0x199											/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/server.go:141
#	0x7fe8bc4b9895	net/http.HandlerFunc.ServeHTTP+0x45																/usr/local/go/src/net/http/server.go:1947
#	0x7fe8bc72a437	github.com/docker/docker/vendor/github.com/gorilla/mux.(*Router).ServeHTTP+0x227										/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/github.com/gorilla/mux/mux.go:103
#	0x7fe8bcb849e1	github.com/docker/docker/api/server.(*routerSwapper).ServeHTTP+0x71												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router_swapper.go:29
#	0x7fe8bc4bc67d	net/http.serverHandler.ServeHTTP+0xbd																/usr/local/go/src/net/http/server.go:2694
#	0x7fe8bc4b88a2	net/http.(*conn).serve+0x652																	/usr/local/go/src/net/http/server.go:1830

1 @ 0x7fe8bc202980 0x7fe8bc212946 0x7fe8bc8b6881 0x7fe8bc8b699d 0x7fe8bc8e259b 0x7fe8bc8e1695 0x7fe8bc8c47d5 0x7fe8bd2e0c06 0x7fe8bd2eda96 0x7fe8bc8c42fb 0x7fe8bc8c4613 0x7fe8bd2a6474 0x7fe8bd2e6976 0x7fe8bd3661c5 0x7fe8bd56842f 0x7fe8bcea7bdb 0x7fe8bcea9f6b 0x7fe8bcb726ca 0x7fe8bcb72b01 0x7fe8bc71c26b 0x7fe8bcb85f4a 0x7fe8bc4b9896 0x7fe8bc72a438 0x7fe8bcb849e2 0x7fe8bc4bc67e 0x7fe8bc4b88a3 0x7fe8bc230711
#	0x7fe8bc8b6880	github.com/docker/docker/vendor/google.golang.org/grpc/transport.(*Stream).waitOnHeader+0x100											/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/google.golang.org/grpc/transport/transport.go:222
#	0x7fe8bc8b699c	github.com/docker/docker/vendor/google.golang.org/grpc/transport.(*Stream).RecvCompress+0x2c											/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/google.golang.org/grpc/transport/transport.go:233
#	0x7fe8bc8e259a	github.com/docker/docker/vendor/google.golang.org/grpc.(*csAttempt).recvMsg+0x63a												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/google.golang.org/grpc/stream.go:515
#	0x7fe8bc8e1694	github.com/docker/docker/vendor/google.golang.org/grpc.(*clientStream).RecvMsg+0x44												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/google.golang.org/grpc/stream.go:395
#	0x7fe8bc8c47d4	github.com/docker/docker/vendor/google.golang.org/grpc.invoke+0x184														/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/google.golang.org/grpc/call.go:83
#	0x7fe8bd2e0c05	github.com/docker/docker/vendor/github.com/containerd/containerd.namespaceInterceptor.unary+0xf5										/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/github.com/containerd/containerd/grpc.go:35
#	0x7fe8bd2eda95	github.com/docker/docker/vendor/github.com/containerd/containerd.(namespaceInterceptor).(github.com/docker/docker/vendor/github.com/containerd/containerd.unary)-fm+0xf5	/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/github.com/containerd/containerd/grpc.go:51
#	0x7fe8bc8c42fa	github.com/docker/docker/vendor/google.golang.org/grpc.(*ClientConn).Invoke+0x10a												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/google.golang.org/grpc/call.go:35
#	0x7fe8bc8c4612	github.com/docker/docker/vendor/google.golang.org/grpc.Invoke+0xc2														/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/google.golang.org/grpc/call.go:60
#	0x7fe8bd2a6473	github.com/docker/docker/vendor/github.com/containerd/containerd/api/services/tasks/v1.(*tasksClient).Start+0xd3								/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/github.com/containerd/containerd/api/services/tasks/v1/tasks.pb.go:421
#	0x7fe8bd2e6975	github.com/docker/docker/vendor/github.com/containerd/containerd.(*process).Start+0xf5												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/github.com/containerd/containerd/process.go:109
#	0x7fe8bd3661c4	github.com/docker/docker/libcontainerd.(*client).Exec+0x4b4															/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/libcontainerd/client_daemon.go:381
#	0x7fe8bd56842e	github.com/docker/docker/daemon.(*Daemon).ContainerExecStart+0xb4e														/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/daemon/exec.go:251
#	0x7fe8bcea7bda	github.com/docker/docker/api/server/router/container.(*containerRouter).postContainerExecStart+0x34a										/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/container/exec.go:125
#	0x7fe8bcea9f6a	github.com/docker/docker/api/server/router/container.(*containerRouter).(github.com/docker/docker/api/server/router/container.postContainerExecStart)-fm+0x6a			/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router/container/container.go:59
#	0x7fe8bcb726c9	github.com/docker/docker/api/server/middleware.ExperimentalMiddleware.WrapHandler.func1+0xd9											/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/middleware/experimental.go:26
#	0x7fe8bcb72b00	github.com/docker/docker/api/server/middleware.VersionMiddleware.WrapHandler.func1+0x400											/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/middleware/version.go:62
#	0x7fe8bc71c26a	github.com/docker/docker/pkg/authorization.(*Middleware).WrapHandler.func1+0x7aa												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/pkg/authorization/middleware.go:59
#	0x7fe8bcb85f49	github.com/docker/docker/api/server.(*Server).makeHTTPHandler.func1+0x199													/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/server.go:141
#	0x7fe8bc4b9895	net/http.HandlerFunc.ServeHTTP+0x45																		/usr/local/go/src/net/http/server.go:1947
#	0x7fe8bc72a437	github.com/docker/docker/vendor/github.com/gorilla/mux.(*Router).ServeHTTP+0x227												/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/vendor/github.com/gorilla/mux/mux.go:103
#	0x7fe8bcb849e1	github.com/docker/docker/api/server.(*routerSwapper).ServeHTTP+0x71														/root/rpmbuild/BUILD/src/engine/.gopath/src/github.com/docker/docker/api/server/router_swapper.go:29
#	0x7fe8bc4bc67d	net/http.serverHandler.ServeHTTP+0xbd																		/usr/local/go/src/net/http/server.go:2694
#	0x7fe8bc4b88a2	net/http.(*conn).serve+0x652																			/usr/local/go/src/net/http/server.go:1830
```

注意，这是一份精简后的docker协程栈信息。从上面的蓝图，我们可以总结出如下结论：

1. 有 717594 个协程被阻塞在docker inspect
2. 有 4175 个协程被阻塞在docker stats
3. 有 1 个协程被阻塞在获取 docker exec的任务ID
4. 有 1 个协程被阻塞在docker exec的执行过程

从上面的结论，我们基本了解了异常容器hang死的原因，在于该容器执行docker exec后未返回(4)，进而导致获取docker exec的任务ID阻塞(3)，由于(3)实现获取了容器锁，进而导致了docker inspect (1)与docker stats (2) 卡死。所以病因并非是docker inspect，而是docker exec。

要想继续往下挖掘，我们现在有必要补充一下背景知识。kubelet启动容器或者在容器内执行命令的完整调用路径如下：

```golang
+--------------------------------------------------------------+
|                                                              |
|   +------------+                                             |
|   |            |                                             |
|   |   kubelet  |                                             |
|   |            |                                             |
|   +------|-----+                                             |
|          |                                                   |
|          |                                                   |
|   +------v-----+       +---------------+                     |
|   |            |       |               |                     |
|   |   dockerd  ------->|  containerd   |                     |
|   |            |       |               |                     |
|   +------------+       +-------|-------+                     |
|                                |                             |
|                                |                             |
|                        +-------v-------+     +-----------+   |
|                        |               |     |           |   |
|                        |containerd-shim----->|   runc    |   |
|                        |               |     |           |   |
|                        +---------------+     +-----------+   |
|                                                              |
+--------------------------------------------------------------+
```

dockerd与containerd可以当做两层nginx代理，containerd-shim是容器的监护人，而runc则是容器启动与命令执行的真正工具人。runc干的事情也非常简单：按照用户指定的配置创建NS，或者进入特定NS，然后执行用户命令。说白了，创建容器就是新建NS，然后在该NS内执行用户指定的命令。

按照上面介绍的背景知识，我们继续往下探索containerd。幸运的是，借助pprof，我们也可以描绘出containerd此时的运行蓝图：

```shell
goroutine profile: total 430
1 @ 0x7f6e55f82740 0x7f6e55f92616 0x7f6e56a8412c 0x7f6e56a83d6d 0x7f6e56a911bf 0x7f6e56ac6e3b 0x7f6e565093de 0x7f6e5650dd3b 0x7f6e5650392b 0x7f6e56b51216 0x7f6e564e5909 0x7f6e563ec76a 0x7f6e563f000a 0x7f6e563f6791 0x7f6e55fb0151
#	0x7f6e56a8412b	github.com/containerd/containerd/vendor/github.com/stevvooe/ttrpc.(*Client).dispatch+0x24b				/go/src/github.com/containerd/containerd/vendor/github.com/stevvooe/ttrpc/client.go:102
#	0x7f6e56a83d6c	github.com/containerd/containerd/vendor/github.com/stevvooe/ttrpc.(*Client).Call+0x15c					/go/src/github.com/containerd/containerd/vendor/github.com/stevvooe/ttrpc/client.go:73
#	0x7f6e56a911be	github.com/containerd/containerd/linux/shim/v1.(*shimClient).Start+0xbe							/go/src/github.com/containerd/containerd/linux/shim/v1/shim.pb.go:1745
#	0x7f6e56ac6e3a	github.com/containerd/containerd/linux.(*Process).Start+0x8a								/go/src/github.com/containerd/containerd/linux/process.go:125
#	0x7f6e565093dd	github.com/containerd/containerd/services/tasks.(*local).Start+0x14d							/go/src/github.com/containerd/containerd/services/tasks/local.go:187
#	0x7f6e5650dd3a	github.com/containerd/containerd/services/tasks.(*service).Start+0x6a							/go/src/github.com/containerd/containerd/services/tasks/service.go:72
#	0x7f6e5650392a	github.com/containerd/containerd/api/services/tasks/v1._Tasks_Start_Handler.func1+0x8a					/go/src/github.com/containerd/containerd/api/services/tasks/v1/tasks.pb.go:624
#	0x7f6e56b51215	github.com/containerd/containerd/vendor/github.com/grpc-ecosystem/go-grpc-prometheus.UnaryServerInterceptor+0xa5	/go/src/github.com/containerd/containerd/vendor/github.com/grpc-ecosystem/go-grpc-prometheus/server.go:29
#	0x7f6e564e5908	github.com/containerd/containerd/api/services/tasks/v1._Tasks_Start_Handler+0x168					/go/src/github.com/containerd/containerd/api/services/tasks/v1/tasks.pb.go:626
#	0x7f6e563ec769	github.com/containerd/containerd/vendor/google.golang.org/grpc.(*Server).processUnaryRPC+0x849				/go/src/github.com/containerd/containerd/vendor/google.golang.org/grpc/server.go:920
#	0x7f6e563f0009	github.com/containerd/containerd/vendor/google.golang.org/grpc.(*Server).handleStream+0x1319				/go/src/github.com/containerd/containerd/vendor/google.golang.org/grpc/server.go:1142
#	0x7f6e563f6790	github.com/containerd/containerd/vendor/google.golang.org/grpc.(*Server).serveStreams.func1.1+0xa0			/go/src/github.com/containerd/containerd/vendor/google.golang.org/grpc/server.go:637
```

同样，我们仅保留了关键的协程信息，从上面的协程栈可以看出，containerd阻塞在接收exec返回结果处，附上关键代码佐证：

```go
func (c *Client) dispatch(ctx context.Context, req *Request, resp *Response) error {
   errs := make(chan error, 1)
   call := &callRequest{
      req:  req,
      resp: resp,
      errs: errs,
   }

   select {
   case c.calls <- call:
   case <-c.done:
      return c.err
   }

   select {        // 此处对应上面协程栈 /go/src/github.com/containerd/containerd/vendor/github.com/stevvooe/ttrpc/client.go:102
   case err := <-errs:
      return filterCloseErr(err)
   case <-c.done:
      return c.err
   }
}
```

containerd将请求传递至containerd-shim之后，一直在等待containerd-shim的返回。

正常情况下，如果我们能够按照调用链路逐个分析每个组件的协程调用栈信息，我们能够很快的定位问题所在。不幸的是，由于线上docker没有开启debug模式，我们无法收集containerd-shim的pprof信息，并且runc也没有开启pprof。因此单纯依赖协程调用链路定位问题这条路被堵死了。

截至目前，我们已经收集了部分关键信息，同时也将问题排查范围更进一步地缩小在containerd-shim与runc之间。接下来我们换一种思路继续排查。

#### 3.2 进程排查

当组件的运行状态无法继续获取时，我们转换一下思维，获取容器的运行状态，也即异常容器此时的进程状态。

既然docker ps执行正常，而docker inspect hang死，首先我们定位异常容器，命令如下：

```shell
docker ps | grep -v NAME | awk '{print $1}' | while read cid; do echo $cid; docker inspect -f {{.State.Pid}} $cid; done
```

拿到异常容器的ID之后，我们就能扫描与该容器相关的所有进程：

```shell
root     11646  6655  0 Jun17 ?        00:01:04 docker-containerd-shim -namespace moby -workdir /home/docker_rt/containerd/daemon/io.containerd.runtime.v1.linux/moby/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
root     11680 11646  0 Jun17 ?        00:00:00 /dockerinit
root     15581 11646  0 Jun17 ?        00:00:00 docker-runc --root /var/run/docker/runtime-runc/moby --log /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/log.json --log-format json exec --process /tmp/runc-process616674997 --detach --pid-file /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/0594c5897a41d401e4d1d7ddd44dacdd316c7e7d53bfdae7f16b0f6b26fcbcda.pid bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5
root     15638 15581  0 Jun17 ?        00:00:00 docker-runc init
```

核心进程列表如上，简单备注下：

- 6655：containerd进程
- 11646：异常容器的containerd-shim进程
- 11680：异常容器的容器启动进程。在容器内查看，因PID NS的隔离，该进程ID是1
- 15581：在异常容器内执行用户命令的进程
- 15638：在异常容器内执行用户命令时，进入容器NS的进程

这里再补充一个背景知识：当我们启动容器时，首先会创建runc init进程，创建并进入新的容器NS；而当我们在容器内执行命令时，首先也会创建runc init进程，进入容器的NS。进入容器的隔离NS中，才会执行用户指定的命令。

面对上面的进程列表，我们无法直观地感受问题究竟由哪个进程引起。因此，我们还需要了解进程当前所处的状态。借助strace，我们逐一展示进程的活动状态：

```shell
// 11646 (container-shim)
Process 11646 attached with 10 threads
[pid 37342] epoll_pwait(5,  <unfinished ...>
[pid 11656] futex(0x818cc0, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 11655] restart_syscall(<... resuming interrupted call ...> <unfinished ...>
[pid 11654] futex(0x818bd8, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 11653] futex(0x7fc730, FUTEX_WAKE, 1 <unfinished ...>
[pid 11651] futex(0xc4200b4148, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 11650] futex(0xc420082948, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 11649] futex(0xc420082548, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 11647] restart_syscall(<... resuming interrupted call ...> <unfinished ...>
[pid 11646] futex(0x7fd008, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 11653] <... futex resumed> )       = 0
[pid 11647] <... restart_syscall resumed> ) = -1 EAGAIN (Resource temporarily unavailable)
[pid 11653] epoll_wait(4,  <unfinished ...>
[pid 11647] pselect6(0, NULL, NULL, NULL, {0, 20000}, 0) = 0 (Timeout)
[pid 11647] futex(0x7fc730, FUTEX_WAIT, 0, {60, 0}


// 11581 (runc exec)
Process 15581 attached with 7 threads
[pid 15619] read(6,  <unfinished ...>
[pid 15592] futex(0xc4200be148, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15591] futex(0x7fd6d25f6238, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15590] futex(0xc420084d48, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15586] futex(0x7fd6d25f6320, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15584] restart_syscall(<... resuming interrupted call ...> <unfinished ...>
[pid 15581] futex(0x7fd6d25d9b28, FUTEX_WAIT, 0, NULL


// 11638 (runc init)
Process 15638 attached with 7 threads
[pid 15648] futex(0x7f512cea5320, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15647] futex(0x7f512cea5238, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15645] futex(0xc4200bc148, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15643] futex(0xc420082d48, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15642] futex(0xc420082948, FUTEX_WAIT, 0, NULL <unfinished ...>
[pid 15639] restart_syscall(<... resuming interrupted call ...> <unfinished ...>
[pid 15638] write(2, "/usr/local/go/src/runtime/proc.g"..., 33
```

从关联进程的活动状态，我们可以得出如下结论：

- runc exec在等待从6号FD读取数据
- runc init在等待从2号FD写入数据

这些FD究竟对应的是什么文件呢？我们借助lsof可以查看：

```shell
// 11638 (runc init)
COMMAND     PID USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
runc:[2:I 15638 root  cwd       DIR               0,41      192 1066743071 /
runc:[2:I 15638 root  rtd       DIR               0,41      192 1066743071 /
runc:[2:I 15638 root  txt       REG                0,4  7644224 1070360467 /memfd:runc_cloned:/proc/self/exe (deleted)
runc:[2:I 15638 root  mem       REG                8,3  2107816    1053962 /usr/lib64/libc-2.17.so
runc:[2:I 15638 root  mem       REG                8,3    19512    1054285 /usr/lib64/libdl-2.17.so
runc:[2:I 15638 root  mem       REG                8,3   266688    1050626 /usr/lib64/libseccomp.so.2.3.1
runc:[2:I 15638 root  mem       REG                8,3   142296    1055698 /usr/lib64/libpthread-2.17.so
runc:[2:I 15638 root  mem       REG                8,3    27168    3024893 /usr/local/gundam/gundam_client/preload/lib64/gundam_preload.so
runc:[2:I 15638 root  mem       REG                8,3   164432    1054515 /usr/lib64/ld-2.17.so
runc:[2:I 15638 root    0r     FIFO                0,8      0t0 1070361745 pipe
runc:[2:I 15638 root    1w     FIFO                0,8      0t0 1070361746 pipe
runc:[2:I 15638 root    2w     FIFO                0,8      0t0 1070361747 pipe
runc:[2:I 15638 root    3u     unix 0xffff881ff8273000      0t0 1070361341 socket
runc:[2:I 15638 root    5u  a_inode                0,9        0       7180 [eventpoll]


// 11581 (runc exec)
COMMAND     PID USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
docker-ru 15581 root  cwd       DIR               0,18      120 1066743076 /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5
docker-ru 15581 root  rtd       DIR                8,3     4096          2 /
docker-ru 15581 root  txt       REG                8,3  7644224     919775 /usr/bin/docker-runc
docker-ru 15581 root  mem       REG                8,3  2107816    1053962 /usr/lib64/libc-2.17.so
docker-ru 15581 root  mem       REG                8,3    19512    1054285 /usr/lib64/libdl-2.17.so
docker-ru 15581 root  mem       REG                8,3   266688    1050626 /usr/lib64/libseccomp.so.2.3.1
docker-ru 15581 root  mem       REG                8,3   142296    1055698 /usr/lib64/libpthread-2.17.so
docker-ru 15581 root  mem       REG                8,3    27168    3024893 /usr/local/gundam/gundam_client/preload/lib64/gundam_preload.so
docker-ru 15581 root  mem       REG                8,3   164432    1054515 /usr/lib64/ld-2.17.so
docker-ru 15581 root    0r     FIFO                0,8      0t0 1070361745 pipe
docker-ru 15581 root    1w     FIFO                0,8      0t0 1070361746 pipe
docker-ru 15581 root    2w     FIFO                0,8      0t0 1070361747 pipe
docker-ru 15581 root    3w      REG               0,18     5456 1066709902 /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/log.json
docker-ru 15581 root    4u  a_inode                0,9        0       7180 [eventpoll]
docker-ru 15581 root    6u     unix 0xffff881ff8275400      0t0 1070361342 socket


// 11646 (container-shim)
COMMAND     PID USER   FD      TYPE             DEVICE SIZE/OFF       NODE NAME
docker-co 11646 root  cwd       DIR               0,18      120 1066743076 /run/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5
docker-co 11646 root  rtd       DIR                8,3     4096          2 /
docker-co 11646 root  txt       REG                8,3  4173632     919772 /usr/bin/docker-containerd-shim
docker-co 11646 root    0r      CHR                1,3      0t0       2052 /dev/null
docker-co 11646 root    1w      CHR                1,3      0t0       2052 /dev/null
docker-co 11646 root    2w      CHR                1,3      0t0       2052 /dev/null
docker-co 11646 root    3r     FIFO                0,8      0t0 1070361745 pipe
docker-co 11646 root    4u  a_inode                0,9        0       7180 [eventpoll]
docker-co 11646 root    5u  a_inode                0,9        0       7180 [eventpoll]
docker-co 11646 root    6u     unix 0xffff881e8cac2800      0t0 1066743079 @/containerd-shim/moby/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/shim.sock
docker-co 11646 root    7u     unix 0xffff881e8cac3400      0t0 1066743968 @/containerd-shim/moby/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/shim.sock
docker-co 11646 root    8r     FIFO                0,8      0t0 1066743970 pipe
docker-co 11646 root    9w     FIFO                0,8      0t0 1070361745 pipe
docker-co 11646 root   10r     FIFO                0,8      0t0 1066743971 pipe
docker-co 11646 root   11u     FIFO               0,18      0t0 1066700778 /run/docker/containerd/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/init-stdout
docker-co 11646 root   12r     FIFO                0,8      0t0 1066743972 pipe
docker-co 11646 root   13w     FIFO               0,18      0t0 1066700778 /run/docker/containerd/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/init-stdout
docker-co 11646 root   14u     FIFO               0,18      0t0 1066700778 /run/docker/containerd/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/init-stdout
docker-co 11646 root   15r     FIFO               0,18      0t0 1066700778 /run/docker/containerd/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/init-stdout
docker-co 11646 root   16u     FIFO               0,18      0t0 1066700779 /run/docker/containerd/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/init-stderr
docker-co 11646 root   17w     FIFO               0,18      0t0 1066700779 /run/docker/containerd/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/init-stderr
docker-co 11646 root   18u     FIFO               0,18      0t0 1066700779 /run/docker/containerd/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/init-stderr
docker-co 11646 root   19r     FIFO               0,18      0t0 1066700779 /run/docker/containerd/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/init-stderr
docker-co 11646 root   20r     FIFO                0,8      0t0 1070361746 pipe
docker-co 11646 root   26r     FIFO                0,8      0t0 1070361747 pipe
```

有心人结合strace与lsof的结果，已经能够自己得出结论：

runc init往2号FD内写数据时阻塞，2号FD对应的类型是pipe类型。而linux pipe有一个默认的数据大小，当写入的数据超过该Size（这个Size可以借助ulimit获取）时，同时读端并未读取数据，写段就会被阻塞。总结一下：containerd-shim启动runc exec去容器内执行用户命令，runc exec启动runc init进入容器时，由于往2号FD写数据超过限制大小而被阻塞。当最底层的runc init被阻塞时，造成了调用链路上所有进程都被阻塞：

```shell
runc init → runc exec → containerd-shim exec → containerd exec → dockerd exec
```

问题定位至此，我们已经了解了docker hang死的原因。但是，现在我们还有如下问题并未解决：

- 为什么runc init会往2号FD (对应go语言的os.Stderr) 中写入超过linux pipe大小限制的数据？
- 为什么runc init出现问题只发生在特定容器？

如果常态下runc init就需要往os.Stdout或者os.Stderr中写入很多数据，那么所有容器的创建都应该有问题。所以，我们可以确定是该异常容器出现了什么未知原因，导致runc init非预期往os.Stderr写入了大量数据。而这被写入的数据就很有可能揭示非预期的异常。

所以，我们需要获取runc init当前正在写入的数据。由于runc init的2号FD是个匿名pipe，我们无法使用常规文件读取的方式获取pipe内的数据。这里感谢鹤哥（本人...）趟坑，找到了一种读取匿名pipe内容的方法：

```shell
# cat /proc/15638/fd/2
runtime/cgo: pthread_create failed: Resource temporarily unavailable
SIGABRT: abort
PC=0x7f512b7365f7 m=0 sigcode=18446744073709551610

goroutine 0 [idle]:
runtime: unknown pc 0x7f512b7365f7
stack: frame={sp:0x7ffe1121a658, fp:0x0} stack=[0x7ffe0ae1bb28,0x7ffe1121ab50)
00007ffe1121a558:  00007ffe1121a6d8  00007ffe1121a6b0
00007ffe1121a568:  0000000000000001  00007f512c527660
00007ffe1121a578:  00007f512c54d560  00007f512c54d208
00007ffe1121a588:  00007f512c333e6f  0000000000000000
00007ffe1121a598:  00007f512c527660  0000000000000005
00007ffe1121a5a8:  0000000000000000  0000000000000001
00007ffe1121a5b8:  00007f512c54d208  00007f512c528000
00007ffe1121a5c8:  00007ffe1121a600  00007f512b704b0c
00007ffe1121a5d8:  00007f512b7110c0  0000000000000000
00007ffe1121a5e8:  00007f512c54d560  00007ffe1121a620
00007ffe1121a5f8:  00007ffe1121a610  000000000f11ed7d
00007ffe1121a608:  00007f512c550153  00000000ffffffff
00007ffe1121a618:  00007f512c550a9b  00007f512b707d00
00007ffe1121a628:  00007f512babc868  00007f512c9e9e5e
00007ffe1121a638:  00007f512d3bb080  00000000000000f1
00007ffe1121a648:  0000000000000011  0000000000000000
00007ffe1121a658: <00007f512b737ce8  0000000000000020
00007ffe1121a668:  0000000000000000  0000000000000000
00007ffe1121a678:  0000000000000000  0000000000000000
00007ffe1121a688:  0000000000000000  0000000000000000
00007ffe1121a698:  0000000000000000  0000000000000000
00007ffe1121a6a8:  0000000000000000  0000000000000000
00007ffe1121a6b8:  0000000000000000  0000000000000000
00007ffe1121a6c8:  0000000000000000  0000000000000000
00007ffe1121a6d8:  0000000000000000  00007f512babc868
00007ffe1121a6e8:  00007f512c9e9e5e  00007f512d3bb080
00007ffe1121a6f8:  00007f512c33f260  00007f512babc1c0
00007ffe1121a708:  00007f512babc1c0  0000000000000001
00007ffe1121a718:  00007f512babc243  00000000000000f1
00007ffe1121a728:  00007f512b7787ec  0000000000000001
00007ffe1121a738:  00007f512babc1c0  000000000000000a
00007ffe1121a748:  00007f512b7e8a4d  000000000000000a
runtime: unknown pc 0x7f512b7365f7
stack: frame={sp:0x7ffe1121a658, fp:0x0} stack=[0x7ffe0ae1bb28,0x7ffe1121ab50)
00007ffe1121a558:  00007ffe1121a6d8  00007ffe1121a6b0
00007ffe1121a568:  0000000000000001  00007f512c527660
00007ffe1121a578:  00007f512c54d560  00007f512c54d208
00007ffe1121a588:  00007f512c333e6f  0000000000000000
00007ffe1121a598:  00007f512c527660  0000000000000005
00007ffe1121a5a8:  0000000000000000  0000000000000001
00007ffe1121a5b8:  00007f512c54d208  00007f512c528000
00007ffe1121a5c8:  00007ffe1121a600  00007f512b704b0c
00007ffe1121a5d8:  00007f512b7110c0  0000000000000000
00007ffe1121a5e8:  00007f512c54d560  00007ffe1121a620
00007ffe1121a5f8:  00007ffe1121a610  000000000f11ed7d
00007ffe1121a608:  00007f512c550153  00000000ffffffff
00007ffe1121a618:  00007f512c550a9b  00007f512b707d00
00007ffe1121a628:  00007f512babc868  00007f512c9e9e5e
00007ffe1121a638:  00007f512d3bb080  00000000000000f1
00007ffe1121a648:  0000000000000011  0000000000000000
00007ffe1121a658: <00007f512b737ce8  0000000000000020
00007ffe1121a668:  0000000000000000  0000000000000000
00007ffe1121a678:  0000000000000000  0000000000000000
00007ffe1121a688:  0000000000000000  0000000000000000
00007ffe1121a698:  0000000000000000  0000000000000000
00007ffe1121a6a8:  0000000000000000  0000000000000000
00007ffe1121a6b8:  0000000000000000  0000000000000000
00007ffe1121a6c8:  0000000000000000  0000000000000000
00007ffe1121a6d8:  0000000000000000  00007f512babc868
00007ffe1121a6e8:  00007f512c9e9e5e  00007f512d3bb080
00007ffe1121a6f8:  00007f512c33f260  00007f512babc1c0
00007ffe1121a708:  00007f512babc1c0  0000000000000001
00007ffe1121a718:  00007f512babc243  00000000000000f1
00007ffe1121a728:  00007f512b7787ec  0000000000000001
00007ffe1121a738:  00007f512babc1c0  000000000000000a
00007ffe1121a748:  00007f512b7e8a4d  000000000000000a

goroutine 1 [running, locked to thread]:
runtime.systemstack_switch()
	/usr/local/go/src/runtime/asm_amd64.s:363 fp=0xc4200a3ed0 sp=0xc4200a3ec8 pc=0x7f512c7281d0
runtime.startTheWorld()
	/usr/local/go/src/runtime/proc.go:978 +0x2f fp=0xc4200a3ee8 sp=0xc4200a3ed0 pc=0x7f512c70221f
runtime.GOMAXPROCS(0x1, 0xc42013d9a0)
	/usr/local/go/src/runtime/debug.go:30 +0xa0 fp=0xc4200a3f10 sp=0xc4200a3ee8 pc=0x7f512c6d9810
main.init.0()
	/go/src/github.com/opencontainers/runc/init.go:14 +0x61 fp=0xc4200a3f30 sp=0xc4200a3f10 pc=0x7f512c992801
main.init()
	<autogenerated>:1 +0x624 fp=0xc4200a3f88 sp=0xc4200a3f30 pc=0x7f512c9a1014
runtime.main()
	/usr/local/go/src/runtime/proc.go:186 +0x1d2 fp=0xc4200a3fe0 sp=0xc4200a3f88 pc=0x7f512c6ff962
runtime.goexit()
	/usr/local/go/src/runtime/asm_amd64.s:2361 +0x1 fp=0xc4200a3fe8 sp=0xc4200a3fe0 pc=0x7f512c72ad71

goroutine 6 [syscall]:
os/signal.signal_recv(0x0)
	/usr/local/go/src/runtime/sigqueue.go:139 +0xa8
os/signal.loop()
	/usr/local/go/src/os/signal/signal_unix.go:22 +0x24
created by os/signal.init.0
	/usr/local/go/src/os/signal/signal_unix.go:28 +0x43

rax    0x0
rbx    0x7f512babc868
rcx    0xffffffffffffffff
rdx    0x6
rdi    0x271
rsi    0x271
rbp    0x7f512c9e9e5e
rsp    0x7ffe1121a658
r8     0xa
r9     0x7f512c524740
r10    0x8
r11    0x206
r12    0x7f512d3bb080
r13    0xf1
r14    0x11
r15    0x0
rip    0x7f512b7365f7
rflags 0x206
cs     0x33
fs     0x0
gs     0x0
exec failed: container_linux.go:348: starting container process caused "read init-p: connection reset by peer"
```

额，runc init因资源不足创建线程失败？？？这种输出显然不是runc的输出，而是go runtime非预期的输出内容。那么资源不足，究竟是什么资源类型资源不足呢？我们在结合 /var/log/message 日志分析：

```shell
Jun 17 03:18:17 host-xx kernel: runc:[2:INIT] invoked oom-killer: gfp_mask=0xd0, order=0, oom_score_adj=997
Jun 17 03:18:17 host-xx kernel: CPU: 14 PID: 12788 Comm: runc:[2:INIT] Tainted: G        W  OE  ------------ T 3.10.0-514.16.1.el7.stable.v1.4.x86_64 #1
Jun 17 03:18:17 host-xx kernel: Hardware name: Inspur SA5212M4/YZMB-00370-107, BIOS 4.1.10 11/14/2016
Jun 17 03:18:17 host-xx kernel: ffff88103841dee0 00000000c4394691 ffff880263e4bcb8 ffffffff8168863d
Jun 17 03:18:17 host-xx kernel: ffff880263e4bd50 ffffffff81683585 ffff88203cc5e300 ffff880ee02b2380
Jun 17 03:18:17 host-xx kernel: 0000000000000001 0000000000000000 0000000000000000 0000000000000046
Jun 17 03:18:17 host-xx kernel: Call Trace:
Jun 17 03:18:17 host-xx kernel: [<ffffffff8168863d>] dump_stack+0x19/0x1b
Jun 17 03:18:17 host-xx kernel: [<ffffffff81683585>] dump_header+0x85/0x27f
Jun 17 03:18:17 host-xx kernel: [<ffffffff81185b06>] ? find_lock_task_mm+0x56/0xc0
Jun 17 03:18:17 host-xx kernel: [<ffffffff81185fbe>] oom_kill_process+0x24e/0x3c0
Jun 17 03:18:17 host-xx kernel: [<ffffffff81093c2e>] ? has_capability_noaudit+0x1e/0x30
Jun 17 03:18:17 host-xx kernel: [<ffffffff811f4d91>] mem_cgroup_oom_synchronize+0x551/0x580
Jun 17 03:18:17 host-xx kernel: [<ffffffff811f41b0>] ? mem_cgroup_charge_common+0xc0/0xc0
Jun 17 03:18:17 host-xx kernel: [<ffffffff81186844>] pagefault_out_of_memory+0x14/0x90
Jun 17 03:18:17 host-xx kernel: [<ffffffff816813fa>] mm_fault_error+0x68/0x12b
Jun 17 03:18:17 host-xx kernel: [<ffffffff81694405>] __do_page_fault+0x395/0x450
Jun 17 03:18:17 host-xx kernel: [<ffffffff816944f5>] do_page_fault+0x35/0x90
Jun 17 03:18:17 host-xx kernel: [<ffffffff81690708>] page_fault+0x28/0x30
Jun 17 03:18:17 host-xx kernel: memory: usage 3145728kB, limit 3145728kB, failcnt 14406932
Jun 17 03:18:17 host-xx kernel: memory+swap: usage 3145728kB, limit 9007199254740988kB, failcnt 0
Jun 17 03:18:17 host-xx kernel: kmem: usage 3143468kB, limit 9007199254740988kB, failcnt 0
Jun 17 03:18:17 host-xx kernel: Memory cgroup stats for /kubepods/burstable/pod6c4333b3-a663-11ea-b39f-6c92bf85beda: cache:0KB rss:0KB rss_huge:0KB mapped_file:0KB swap:0KB inactive_anon:0KB active_anon:0KB inactive_file:0KB active_file:0KB unevictable:0KB
Jun 17 03:18:17 host-xx kernel: Memory cgroup stats for /kubepods/burstable/pod6c4333b3-a663-11ea-b39f-6c92bf85beda/b761e05249245695278b3f409d2d6e5c6a5bff6995ff0cf44d03af4aa9764a30: cache:0KB rss:40KB rss_huge:0KB mapped_file:0KB swap:0KB inactive_anon:0KB active_anon:40KB inactive_file:0KB active_file:0KB unevictable:0KB
Jun 17 03:18:17 host-xx kernel: Memory cgroup stats for /kubepods/burstable/pod6c4333b3-a663-11ea-b39f-6c92bf85beda/1d1750ecc627cc5d60d80c071b2eb4d515ee8880c5b5136883164f08319869b0: cache:0KB rss:0KB rss_huge:0KB mapped_file:0KB swap:0KB inactive_anon:0KB active_anon:0KB inactive_file:0KB active_file:0KB unevictable:0KB
Jun 17 03:18:17 host-xx kernel: Memory cgroup stats for /kubepods/burstable/pod6c4333b3-a663-11ea-b39f-6c92bf85beda/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5: cache:0KB rss:2220KB rss_huge:0KB mapped_file:0KB swap:0KB inactive_anon:0KB active_anon:2140KB inactive_file:0KB active_file:0KB unevictable:0KB
Jun 17 03:18:17 host-xx kernel: Memory cgroup stats for /kubepods/burstable/pod6c4333b3-a663-11ea-b39f-6c92bf85beda/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5/super-agent: cache:0KB rss:0KB rss_huge:0KB mapped_file:0KB swap:0KB inactive_anon:0KB active_anon:0KB inactive_file:0KB active_file:0KB unevictable:0KB
Jun 17 03:18:17 host-xx kernel: [ pid ]   uid  tgid total_vm      rss nr_ptes swapents oom_score_adj name
Jun 17 03:18:17 host-xx kernel: [30598]     0 30598      255        1       4        0          -998 pause
Jun 17 03:18:17 host-xx kernel: [11680]     0 11680   164833     1118      20        0           997 dockerinit
Jun 17 03:18:17 host-xx kernel: [12788]     0 12788   150184     1146      23        0           997 runc:[2:INIT]
Jun 17 03:18:17 host-xx kernel: oom-kill:,cpuset=bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5,mems_allowed=0-1,oom_memcg=/kubepods/burstable/pod6c4333b3-a663-11ea-b39f-6c92bf85beda,task_memcg=/kubepods/burstable/pod6c4333b3-a663-11ea-b39f-6c92bf85beda/bbd5e4b5f9c13666dd0ec7ff7afb2c4c2b0ede40a4adf1de43cc31c606f283f5,task=runc:[2:INIT],pid=12800,uid=0
Jun 17 03:18:17 host-xx kernel: Memory cgroup out of memory: Kill process 12800 (runc:[2:INIT]) score 997 or sacrifice child
Jun 17 03:18:17 host-xx kernel: Killed process 12788 (runc:[2:INIT]) total-vm:600736kB, anon-rss:3296kB, file-rss:276kB, shmem-rss:1012kB
```

在 /var/log/message 中可以找到该容器在大约1个月前大量的OOM日志记录，同时时间也基本匹配。

所以总结下，在一个非常关键的时间节点，runc init由于内存资源不足，创建线程失败，触发go runtime的非预期输出，进而造成runc init阻塞在写pipe操作。

定位至此，问题的全貌已经基本描述清楚。但是我们还有一个疑问，既然runc init再往pipe中写数据，难道没有其他进程来读取这个内容吗？

大家还记得上面lsof执行的结果吗？有心人一定发现了该pipe的读端是谁了，对，就是containerd-shim，对应的pipe的inode编号为1070361747。那么，为什么containerd-shim没有来读pipe里面的内容呢？我们结合代码来分析：

```go
func (e *execProcess) start(ctx context.Context) (err error) {
   ......
   if err := e.parent.runtime.Exec(ctx, e.parent.id, e.spec, opts); err != nil {   // 执行runc init
      close(e.waitBlock)
      return e.parent.runtimeError(err, "OCI runtime exec failed")
   }
   ......
   else if !e.stdio.IsNull() {
      fifoCtx, cancel := context.WithTimeout(ctx, 15*time.Second)
      defer cancel()

      if err := copyPipes(fifoCtx, e.io, e.stdio.Stdin, e.stdio.Stdout, e.stdio.Stderr, &e.wg, &copyWaitGroup); err != nil {   // 读pipe
         return errors.Wrap(err, "failed to start io pipe copy")
      }
   }
   ......
}


func (r *Runc) Exec(context context.Context, id string, spec specs.Process, opts *ExecOpts) error {
   ......
   cmd := r.command(context, append(args, id)...)
   if opts != nil && opts.IO != nil {
      opts.Set(cmd)
   }
   ......
   ec, err := Monitor.Start(cmd)
   ......
   status, err := Monitor.Wait(cmd, ec)
   ......
}
```

额，containerd-shim的设计是，等待runc init执行完成之后，再来读取pipe中的内容。但是此时的runc init由于非预期的写入数据量比较大，被阻塞在了写pipe操作处。。。完美的死锁。

终于，本次docker hang死问题的核心脉络都已清楚。接下来我们聊聊怎么解决方案。

### 4. 解决方案

当大家了解了docker hang死的成因之后，我们可以针对性的提出如下解决办法。

#### 4.1 最直观的办法

既然docker exec可能会引起docker hang死，那么我们禁用系统中所有的docker exec操作即可。最典型的是kubelet的probe，当前我们默认给所有Pod添加了ReadinessProbe，并且是以exec的形式进入容器内执行命令。我们调整kubelet的探测行为，修改为tcp或者http probe即可。

这里虽然改动不大，但是涉及业务容器的改造成本太大了，如何迁移存量集群是个大问题。

#### 4.2 最根本的办法

既然当前containerd-shim读pipe需要等待runc exec执行完毕，如果我们将读pipe的操作提前至runc exec命令执行之前，理论上也可以避免死锁。

同样。这种方案的升级成本太高了，升级containerd-shim时需要重启存量的所有容器，这个方案基本不可能通过。

#### 4.3 最简单的办法

既然runc init阻塞在写pipe，我们主动读取pipe内的内容，也能让runc init顺利推出。

再将本解决方案自动化的过程中，如何能够识别如docker hang死是由于写pipe导致的，是一个小小的挑战。但是相对于以上两种解决方案，我认为还是值得一试，毕竟影响面微乎其微。

### 5. 后续

其实我们在读pipe的时候还引起了一个其他的问题，后续我再来为大家介绍。

另外，docker hang死的原因远飞这一种，本次排查的结果也并非适用于所有场景。希望各位看官能够根据自己的现场排查问题。

本次docker hang死的排查之旅已然告终。

本次排查由四人小分队 @飞哥 @鹤哥（本人） @博哥 @我 （培龙） 一起排查长达数天的结论
