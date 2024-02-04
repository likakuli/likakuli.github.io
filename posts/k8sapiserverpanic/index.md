# kube-apiserver 又 Panic 了 - handler


Kube-apiserver 不止 [OOM](https://mp.weixin.qq.com/s/FhNl1c2BdXJK-sDb0qzahw)，还可能 Panic。

# 现象

Kube-apiserver 在处理外部请求时发生不可恢复的报错，直接 Fatal 退出运行。看日志调用堆栈，会发现 `concurrent map iteration and map write` 的字样。这是 golang 检测到有对 map 的并发读写后返回的一个 fatal 报错的内容，无法通过 recover 捕捉并恢复。

# 影响版本

目前线上使用的主流版本，都有这个问题，最新的 master（截止写这篇的时间） 分支依然有这个问题。虽然几乎所有版本中都存在这个问题，但由于其触发条件有一定的要求，也和参数配置有关，所以大家可能遇到的次数不多，至少相比 OOM 问题遇到的次数会少一些。

# 触发原因

这个问题的触发条件是 api 请求超时，同时在执行后续 filter 时也用到了 response header。

Kube-apiserver 在启动时通过 `DefaultBuildHandlerChain` 注册了特别多的 filter/handler，其中就有一个专门用来控制请求超时的 timeoutHandler。kube-apiserver 并没有为单独的 handler 或者对应的 Goroutine 设置一个 WriteTimeout 的超时，而是单纯的通过 timeoutHandler 实现全部 NonLongRunning 类型请求的超时控制。

```go
func DefaultBuildHandlerChain(apiHandler http.Handler, c *Config) http.Handler {
	handler := apiHandler

	handler = filterlatency.TrackCompleted(handler)
	handler = genericapifilters.WithAuthorization(handler, c.Authorization.Authorizer, c.Serializer)
	handler = filterlatency.TrackStarted(handler, c.TracerProvider, "authorization")

	if c.FlowControl != nil {
		workEstimatorCfg := flowcontrolrequest.DefaultWorkEstimatorConfig()
		requestWorkEstimator := flowcontrolrequest.NewWorkEstimator(
			c.StorageObjectCountTracker.Get, c.FlowControl.GetInterestedWatchCount, workEstimatorCfg, c.FlowControl.GetMaxSeats)
		handler = filterlatency.TrackCompleted(handler)
		handler = genericfilters.WithPriorityAndFairness(handler, c.LongRunningFunc, c.FlowControl, requestWorkEstimator, c.RequestTimeout/4)
		handler = filterlatency.TrackStarted(handler, c.TracerProvider, "priorityandfairness")
	} else {
		handler = genericfilters.WithMaxInFlightLimit(handler, c.MaxRequestsInFlight, c.MaxMutatingRequestsInFlight, c.LongRunningFunc)
	}

	handler = filterlatency.TrackCompleted(handler)
	handler = genericapifilters.WithImpersonation(handler, c.Authorization.Authorizer, c.Serializer)
	handler = filterlatency.TrackStarted(handler, c.TracerProvider, "impersonation")

	handler = filterlatency.TrackCompleted(handler)
	handler = genericapifilters.WithAudit(handler, c.AuditBackend, c.AuditPolicyRuleEvaluator, c.LongRunningFunc)
	handler = filterlatency.TrackStarted(handler, c.TracerProvider, "audit")

	failedHandler := genericapifilters.Unauthorized(c.Serializer)
	failedHandler = genericapifilters.WithFailedAuthenticationAudit(failedHandler, c.AuditBackend, c.AuditPolicyRuleEvaluator)

	failedHandler = filterlatency.TrackCompleted(failedHandler)
	handler = filterlatency.TrackCompleted(handler)
	handler = genericapifilters.WithAuthentication(handler, c.Authentication.Authenticator, failedHandler, c.Authentication.APIAudiences, c.Authentication.RequestHeaderConfig)
	handler = filterlatency.TrackStarted(handler, c.TracerProvider, "authentication")

	handler = genericfilters.WithCORS(handler, c.CorsAllowedOriginList, nil, nil, nil, "true")

	// WithTimeoutForNonLongRunningRequests will call the rest of the request handling in a go-routine with the
	// context with deadline. The go-routine can keep running, while the timeout logic will return a timeout to the client.
	handler = genericfilters.WithTimeoutForNonLongRunningRequests(handler, c.LongRunningFunc)

	handler = genericapifilters.WithRequestDeadline(handler, c.AuditBackend, c.AuditPolicyRuleEvaluator,
		c.LongRunningFunc, c.Serializer, c.RequestTimeout)
	handler = genericfilters.WithWaitGroup(handler, c.LongRunningFunc, c.NonLongRunningRequestWaitGroup)
	if c.ShutdownWatchTerminationGracePeriod > 0 {
		handler = genericfilters.WithWatchTerminationDuringShutdown(handler, c.lifecycleSignals, c.WatchRequestWaitGroup)
	}
	if c.SecureServing != nil && !c.SecureServing.DisableHTTP2 && c.GoawayChance > 0 {
		handler = genericfilters.WithProbabilisticGoaway(handler, c.GoawayChance)
	}
	handler = genericapifilters.WithWarningRecorder(handler)
	handler = genericapifilters.WithCacheControl(handler)
	handler = genericfilters.WithHSTS(handler, c.HSTSDirectives)
	if c.ShutdownSendRetryAfter {
		handler = genericfilters.WithRetryAfter(handler, c.lifecycleSignals.NotAcceptingNewRequest.Signaled())
	}
	handler = genericfilters.WithHTTPLogging(handler)
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerTracing) {
		handler = genericapifilters.WithTracing(handler, c.TracerProvider)
	}
	handler = genericapifilters.WithLatencyTrackers(handler)
	handler = genericapifilters.WithRequestInfo(handler, c.RequestInfoResolver)
	handler = genericapifilters.WithRequestReceivedTimestamp(handler)
	handler = genericapifilters.WithMuxAndDiscoveryComplete(handler, c.lifecycleSignals.MuxAndDiscoveryComplete.Signaled())
	handler = genericfilters.WithPanicRecovery(handler, c.RequestInfoResolver)
	handler = genericapifilters.WithAuditInit(handler)
	return handler
}
```

golang http handler 经典模式，handler 最后的执行顺序与注册顺序相反，也就是说请求最终会先进入到最后通过 `WithAuditInit` 注册的 handler 里面，然后再进入到 `WithPanicRecovery` 注册的 handler 里面，最终经过链式的传递，请求会被真正的 apiHandler 处理，这才是处理真正业务逻辑的入口。

# 万恶之源

在这一坨的 handler 里面，万恶之源是 `WithTimeoutForNonLongRunningRequests` 返回的 timeoutHandler。

```go
func (t *timeoutHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	r, longRunning, postTimeoutFn, err := t.timeout(r)
	if longRunning {
		t.handler.ServeHTTP(w, r)
		return
	}

	timeoutCh := r.Context().Done()

	// resultCh is used as both errCh and stopCh
	resultCh := make(chan interface{})
	var tw timeoutWriter
	tw, w = newTimeoutWriter(w)

	// Make a copy of request and work on it in new goroutine
	// to avoid race condition when accessing/modifying request (e.g. headers)
	rCopy := r.Clone(r.Context())
	go func() {
		defer func() {
			err := recover()
			// do not wrap the sentinel ErrAbortHandler panic value
			if err != nil && err != http.ErrAbortHandler {
				// Same as stdlib http server code. Manually allocate stack
				// trace buffer size to prevent excessively large logs
				const size = 64 << 10
				buf := make([]byte, size)
				buf = buf[:runtime.Stack(buf, false)]
				err = fmt.Sprintf("%v\n%s", err, buf)
			}
			resultCh <- err
		}()
		t.handler.ServeHTTP(w, rCopy)
	}()
	select {
	case err := <-resultCh:
		// panic if error occurs; stop otherwise
		if err != nil {
			panic(err)
		}
		return
	case <-timeoutCh:
		defer func() {
			// resultCh needs to have a reader, since the function doing
			// the work needs to send to it. This is defer'd to ensure it runs
			// ever if the post timeout work itself panics.
			go func() {
				timedOutAt := time.Now()
				res := <-resultCh

				status := metrics.PostTimeoutHandlerOK
				if res != nil {
					// a non nil res indicates that there was a panic.
					status = metrics.PostTimeoutHandlerPanic
				}

				metrics.RecordRequestPostTimeout(metrics.PostTimeoutSourceTimeoutHandler, status)
				err := fmt.Errorf("post-timeout activity - time-elapsed: %s, %v %q result: %v",
					time.Since(timedOutAt), r.Method, r.URL.Path, res)
				utilruntime.HandleError(err)
			}()
		}()
		httplog.SetStacktracePredicate(r.Context(), func(status int) bool {
			return false
		})
		defer postTimeoutFn()
		tw.timeout(err)
	}
}
```

可以看到 **timeoutHandler 超时处理并没有终止 `t.handler` 的执行**，`t.handler.ServeHTTP` 在一个单独的 Goroutine 里面。由于 HTTP2/ping 特性的存在，如果客户端已经断开连接，刚提到的 Goroutine 最终是可以被释放的。

timeoutHandler 在处理请求时会用到实现自定义 timeoutWriter 接口的 baseTimeoutWriter，timeoutWriter 对外部传入的 ResponseWriter 做了封装，最终在 `t.handler.ServeHTTP` 传入的是经过封装后的 ResponseWriter，如下

```go
type baseTimeoutWriter struct {
	w http.ResponseWriter

	// headers written by the normal handler
	handlerHeaders http.Header

	mu sync.Mutex
	// if the timeout handler has timeout
	timedOut bool
	// if this timeout writer has wrote header
	wroteHeader bool
	// if this timeout writer has been hijacked
	hijacked bool
}

func newTimeoutWriter(w http.ResponseWriter) (timeoutWriter, http.ResponseWriter) {
	base := &baseTimeoutWriter{w: w, handlerHeaders: w.Header().Clone()}
	wrapped := responsewriter.WrapForHTTP1Or2(base)

	return base, wrapped
}
```

baseTimeoutWriter 包含 sync.Mutex，在其实现的 ResponseWriter 接口中都对操作进行了加锁，可以避免 timeoutHandler 与**注册位置在其前面的其他 handler 直接操作 ResponseWriter** 时的 race condition 问题，因为传递给其他 handler 的 ResponseWriter 已经是 baseTimeoutWriter 了。

除此之外还存在着其他 race condition，比如：

1. 注册在 timeoutHandler 之后的其他 handler 如果有对 ResponseWriter header 的操作；
2. 其他 handler 有对 ResponseWriter header 的读操作；

# 问题进展

## 最新状态

问题出现的根本原因在于上面提到的 `t.handler.ServeHTTP` 与 timeoutHandler 并行执行，可能出现对 Response Header 的并发读写或者并发写。所以从根源上解决此问题的办法是避免 handler 之间的并行执行，也就是 timeout 要同时停止 `t.handler.ServeHTTP` 的执行，这就依赖 golang 本身的实现了。在 golang v1.20 中实现了对 per handler 的超时控制，详情可以参考 [golang/go#54136](https://github.com/golang/go/issues/54136)，k8s 侧的实现参考 [set per handler read/write deadline](https://github.com/kubernetes/kubernetes/pull/114189)，此 PR 于 2022 年 11 月份提交，至今尚未合并到 master 分支上。

也就是说问题尚未彻底解决，在使用过程中仍然可能会遇到 header race condition 导致的 panic 问题，但已经提到的一些很具体的 case 已经得到了解决。

## 历史修复

[Fix race condition in logging when request times out](https://github.com/kubernetes/kubernetes/pull/105734) 修复了请求超时后 respLogger handler 打印日志时从 request header 中读取 UserAgent 的操作与 t.handler.ServeHTTP 执行后其他 handler 操作 request header 的 race condition。但是这个修改并不彻底，respLogger 里面还存在其他使用 request header 的地方，比如获取 verb 时最后会调用 CleanVerb 函数，他也会访问 request header。

[Fix header mutation race in timeout filter](https://github.com/kubernetes/kubernetes/pull/107452) 在构建 baseTimeoutWriter 时对传入的 ResponseWriter 的 header 进行了复制保存为 handlerHeaders，然后在其实现的 ResponseWriter 接口的 Header() 中返回 handlerHeaders，同时在 Write() 方法中也是遍历了 handlerHeaders 写到最终的 ResponseHeader 里面。这个 PR 可以避免注册在 timeoutHandler 之前的 handler 与 timeoutWriter 的 timeout() 方法对 ResponseWriter header 的并发写操作导致的问题。

[Copy request in timeout handler](https://github.com/kubernetes/kubernetes/pull/108455) 在 timeoutHandler 里面 Clone 了外部传进来的 request 对象，把新的对象传递给 `t.handler.ServeHTTP` 方法，来彻底解决 timeoutHandler 之后注册的 handler 中使用到了 request header 而导致的 race condition 问题，从最开始的代码片段中可以看到这个操作。

[apiserver: warning.AddWarning should not panic when request times out](https://github.com/kubernetes/kubernetes/pull/115282) 通过调整 `WithWarningRecorder` 的顺序到 `WithTimeoutForNonLongRunningRequests` 前面，解决了 ResponseWriter header 并发写导致 Panic 的一个特例。放在前面后 recorder 里面使用的 ResponseWriter 就是 baseTimeoutWriter 了，其对 WriteHeader() 方法的调用加了锁，就可以避免这个问题。recorder 在注册 handler 时只是被写到了 context 里面，真正被使用的地方很多，其中就有 `WithAuthentication` 返回的 handler。

# 总结

因为 request 或者 response header 的并发读写导致的 panic 问题至今尚未完全解决，如果遇到的话可以在报错堆栈里面搜一下看是否存在 Header 和 handler 关键字，如果有，那么基本就八九不离十可以确定问题了。

理论上目前因 request header 导致的问题已经被彻底修复，一般情况下服务端也很少去修改 request header。等 set per handler read/write deadline 这个 PR 合入主干后，理论上因为 response header 导致的问题也可以被彻底修复。

kube-apiserver Panic 的一个影响是上面的所有请求可能与其他实例建立连接，如果量比较大，而 k8s 服务端本身又没有做好限流保护的话，很可能会导致其他实例出现 OOM，导致 k8s 控制面雪崩。








