# Knative通过header访问指定版本


### 背景

> knative 0.14.0
>
> 实际修改可能与贴出来的代码不符，贴出来的代码只是为了方便快速实现功能

最近在搭建公司级的serverless平台，需要用到域名来访问内部服务，采取的是通过PATH来区分不同的服务，域名采用同一个。上一篇已经解决了通过Path访问不同服务的问题，但是在灰度过程中可能会想测试下新版本时候正常，如何将流量打到指定版本上呢？原生的knative是通过url的不同实现的，可以配置一个根据版本生成url的模板，设置后不同版本的服务url不同。但是我们的场景是所有服务url相同，于是我们约定通过在设置特殊的header的来实现此功能

### 方案

原生通过url来区分不同版本，实现方式是通过在生成vs时，设置其Match的条件Authroty为对应的url prefix即可。显然无法满足我们当前统一使用一个url的场景。但是我们可以参考其实现方式，换一个维度，靠header实现即可，但是又不能影响正常访问，即不添加header的时候，流量按照设置的比例打到不同的revision上，添加了header后，需要将流量打到指定版本，所以不能简单的在Match中添加Header，**需要分别设置正常访问的情况和访问指定版本的情况，且访问指定版本的配置应该顺序靠前**。

```go
// MakeIngressSpec creates a new IngressSpec
func MakeIngressSpec(
	ctx context.Context,
	r *servingv1.Route,
	tls []v1alpha1.IngressTLS,
	targets map[string]traffic.RevisionTargets,
	visibility map[string]netv1alpha1.IngressVisibility,
	acmeChallenges ...v1alpha1.HTTP01Challenge,
) (v1alpha1.IngressSpec, error) {
	...

	// add custom external domains
	customHostStr := r.Annotations["serverless.didichuxing.com/domains"]
  // 倒序，否则不生效，因为访问指定版本时name不为空，不区分版本时name默认为空
	sort.Sort(sort.Reverse(sort.StringSlice(names)))
	if len(customHostStr) > 0 {
		customHosts := strings.Split(customHostStr, ";")
		for _, name := range names {
			if name != "default" {
				visibility := netv1alpha1.IngressVisibilityExternalIP
				rule := *makeIngressRule(customHosts, r.Namespace, visibility, name, targets[name])
				// If this is a public rule, we need to configure ACME challenge paths.
				rule.HTTP.Paths = append(
					makeACMEIngressPaths(challengeHosts, customHosts), rule.HTTP.Paths...)
				rules = append(rules, rule)
			}
		}
	}

	...
}

func makeIngressRule(domains []string, ns string, visibility netv1alpha1.IngressVisibility, name string, targets traffic.RevisionTargets) *v1alpha1.IngressRule {
	...

	return &v1alpha1.IngressRule{
		Hosts:      domains,
		Visibility: visibility,
		HTTP: &v1alpha1.HTTPIngressRuleValue{
			Paths: []v1alpha1.HTTPIngressPath{{
				Splits: splits,
				// TODO(lichuqiang): #2201, plumbing to config timeout and retries.
        // 把tag name保存下来，传递给vs，用来区分是否需要设置header
				AppendHeaders: map[string]string{
					"RevisionName": name,
				},
			}},
		},
	}
}

func makeVirtualServiceRoute(hosts sets.String, usn string, http *v1alpha1.HTTPIngressPath, gateways map[v1alpha1.IngressVisibility]sets.String, visibility v1alpha1.IngressVisibility) *istiov1alpha3.HTTPRoute {
	...

	// add revision tag header to custom domain
  // 获取传递过来的tag，设置match header
	if tag := http.AppendHeaders["RevisionName"]; tag != "" {
		for i := 0; i < len(matches); i++ {
			if matches[i].Headers == nil {
				matches[i].Headers = make(map[string]*istiov1alpha3.StringMatch)
			}
			matches[i].Headers["RevisionName"] = &istiov1alpha3.StringMatch{
				MatchType: &istiov1alpha3.StringMatch_Exact{
					Exact: http.Splits[0].ServiceName,
				},
			}
		}
	}

	...
}
```

### 总结

至此，已经实现了通过统一域名访问集群内服务，且根据Path转发请求，并且可以通过在访问时添加指定的header来把流量打到指定版本上，这在灰度或者测试时是一个非常实用的功能。
