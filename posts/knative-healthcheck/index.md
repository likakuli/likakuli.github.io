# Knative健康检查


### 背景

> knative 0.14.0
>
> 实际修改可能与贴出来的代码不符，贴出来的代码只是为了方便快速实现功能

在支持了前面的定制功能后，集群中部署ksvc服务时会报IngressNotConfigured错误

### 原因分析

首先根据错误提示及日志信息，可以发现是在做健康检查的时候出的问题，期望得到200，但是得到了404

```go
func (m *Prober) probeVerifier(item *workItem) prober.Verifier {
	return func(r *http.Response, _ []byte) (bool, error) {
		// In the happy path, the probe request is forwarded to Activator or Queue-Proxy and the response (HTTP 200)
		// contains the "K-Network-Hash" header that can be compared with the expected hash. If the hashes match,
		// probing is successful, if they don't match, a new probe will be sent later.
		// An HTTP 404/503 is expected in the case of the creation of a new Knative service because the rules will
		// not be present in the Envoy config until the new VirtualService is applied.
		// No information can be extracted from any other scenario (e.g. HTTP 302), therefore in that case,
		// probing is assumed to be successful because it is better to say that an Ingress is Ready before it
		// actually is Ready than never marking it as Ready. It is best effort.
		switch r.StatusCode {
		case http.StatusOK:
			hash := r.Header.Get(network.HashHeaderName)
			switch hash {
			case "":
				m.logger.Errorf("Probing of %s abandoned, IP: %s:%s: the response doesn't contain the %q header",
					item.url, item.podIP, item.podPort, network.HashHeaderName)
				return true, nil
			case item.ingressState.hash:
				return true, nil
			default:
				m.logger.Warnf("unexpected hash: want %q, got %q", item.ingressState.hash, hash)
				return true, nil
			}
    // 日志中报错的地方，探活希望得到200，但是得到了404
		case http.StatusNotFound, http.StatusServiceUnavailable:
			return false, fmt.Errorf("unexpected status code: want %v, got %v", http.StatusOK, http.StatusNotFound)
		default:
			m.logger.Errorf("Probing of %s abandoned, IP: %s:%s: the response status is %v, expected 200 or 404",
				item.url, item.podIP, item.podPort, r.StatusCode)
			return true, nil
		}
	}
}

```

其实这时候大致也能猜到是什么原因了，因为我们定制了通过USN进行过滤，探活的时候，Url中其实是没有USN的。下一步就是顺藤摸瓜，找到探活对应的代码验证我们的猜想，也比较简单

```go
// processWorkItem processes a single work item from workQueue.
// It returns false when there is no more items to process, true otherwise.
func (m *Prober) processWorkItem() bool {
	...
  
  // probePath /healthz
	probeURL.Path = path.Join(probeURL.Path, probePath)

	ok, err := prober.Do(
		item.context,
		transport,
		probeURL.String(),
		prober.WithHeader(network.UserAgentKey, network.IngressReadinessUserAgent),
		prober.WithHeader(network.ProbeHeaderName, network.ProbeHeaderValue),
		m.probeVerifier(item))

	...
}

```

可以看到探活的时候就是拿Path拼上/heathz，验证了我们的猜想

### 修复

修改也就比较简单了，在添加wotkitem时，预先把USN添加到path中即可

```go
func (l *gatewayPodTargetLister) ListProbeTargets(ctx context.Context, ing *v1alpha1.Ingress) ([]status.ProbeTarget, error) {
	...
			// Use sorted hosts list for consistent ordering.
			for i, host := range gatewayHosts[gatewayName].List() {
				newURL := *target.URLs[0]
				newURL.Host = host + ":" + target.Port
				var usn string
				if ing.Annotations != nil {
					usn = ing.Annotations["serverless.didichuxing.com/usn"]
				}
				newURL.Path = path.Join(newURL.Path, usn)
				qualifiedTarget.URLs[i] = &newURL
  ...
}
```

### 总结

通过这个问题也看到了对于一些细节和关键流程掌握的还不够，还是需要进行系统性的学习。至于健康检查的逻辑，和k8s的健康检查稍有不同，参考[这篇文章](https://zhuanlan.zhihu.com/p/88459310)
