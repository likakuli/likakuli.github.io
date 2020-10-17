# Knative根据Path转发请求


### 背景

> knative 0.14.0
>
> 实际修改可能与贴出来的代码不符，贴出来的代码只是为了方便快速实现功能

最近在搭建公司级的serverless平台，需要用到域名来访问内部服务，采取的是通过PATH来区分不同的服务，域名采用同一个。这与原生knative的设计存在差异，原生的做法是每个服务一个自己的域名，通过域名把流量打到不同的服务上，我们已经在[上一篇](http://likakuli.com/post/2020/07/09/knative_ingress_gateway/)中解决了自定义域名无法访问knative集群的问题，这一篇来解决如何通过不同的Path访问到不同的服务

### 方案

两个问题需要我们来解决：

1. 不同服务的Path可能相同，如何区分
2. 原生通过ksvc的方式不支持设置Path（通过自己创建各种类型的资源可以实现，但是控制比较复杂，而且上层需要修改适配）

解决方案：

1. 每个服务一个USN，使用USN作为唯一标识
2. 修改knative，支持通过Path访问
3. 转发后需要rewrite url，把USN去掉，因为业务代码中的路由里不可能包含USN

其中第一点不需要代码改动，我们主要来实现第二、三点。

vs本身是支持根据Path转发的功能的，但是并没有在ksvc中暴露出来，所以我们需要在king创建vs的时候动态注入进去，同时在destination中添加url rewrite的逻辑。

```go
func makeVirtualServiceSpec(ing *v1alpha1.Ingress, gateways map[v1alpha1.IngressVisibility]sets.String, hosts sets.String) *istiov1alpha3.VirtualService {
	spec := istiov1alpha3.VirtualService{
		Hosts: hosts.List(),
	}

  // 自定义功能
	usn := ing.Annotations["serverless.didichuxing.com/usn"]
	if usn != "" {
		usn = "/" + strings.Trim(usn, "/") + "/"
	}

	gw := sets.String{}
	for _, rule := range ing.Spec.Rules {
		for _, p := range rule.HTTP.Paths {
			hosts := hosts.Intersection(sets.NewString(rule.Hosts...))
			if hosts.Len() != 0 {
				http := makeVirtualServiceRoute(hosts, usn, &p, gateways, rule.Visibility)
				// Add all the Gateways that exist inside the http.match section of
				// the VirtualService.
				// This ensures that we are only using the Gateways that actually appear
				// in VirtualService routes.
				for _, m := range http.Match {
					gw = gw.Union(sets.NewString(m.Gateways...))
				}
				// rewrite path，重定向，消除USN
				if usn != "" {
					if http.Rewrite == nil {
						http.Rewrite = &istiov1alpha3.HTTPRewrite{}
					}
					http.Rewrite.Uri = "/"
				}
				spec.Http = append(spec.Http, http)
			}
		}
	}

	spec.Gateways = gw.List()
	return &spec
}

func makeVirtualServiceRoute(hosts sets.String, usn string, http *v1alpha1.HTTPIngressPath, gateways map[v1alpha1.IngressVisibility]sets.String, visibility v1alpha1.IngressVisibility) *istiov1alpha3.HTTPRoute {
	...
  
	for _, host := range hosts.List() {
		g := gateways[visibility]
		if strings.HasSuffix(host, clusterDomainName) && len(gateways[v1alpha1.IngressVisibilityClusterLocal]) > 0 {
			// For local hostname, always use private gateway
			g = gateways[v1alpha1.IngressVisibilityClusterLocal]
		}
		matches = append(matches, makeMatch(host, usn, http.Path, g)...)
	}
  
  ...
}

func makeMatch(host string, usn string, pathRegExp string, gateways sets.String) []*istiov1alpha3.HTTPMatchRequest {
	...
  
		// add custom usn filter，添加USN的过滤条件
		if usn != "" {
			if i == 0 {
				matches[i].Uri = &istiov1alpha3.StringMatch{
					MatchType: &istiov1alpha3.StringMatch_Prefix{Prefix: usn},
				}
			} else {
				matches[i].Uri = &istiov1alpha3.StringMatch{
					MatchType: &istiov1alpha3.StringMatch_Exact{Exact: strings.TrimRight(usn, "/")},
				}
			}

		}

		...
}
```

修改比较简单，完全就是按照之前说的两点进行的。其中有一个比较tricky的地方就是实现url rewrite的方式，因为社区中的vs（istio里的crd）其实是存在问题的，我们为了规避这个问题，特意做了一些特殊设置。参考[这里](https://github.com/istio/istio/issues/8076)，大致意思就是目前vs不支持url rewrite为空，rewrite为空之后，实际访问的时候需要在url的最后加上/，否则会返回400，但是我们很多前端网站主页就是一个域名，后面不跟任何内容，那这时候就有问题了，总不能再告诉用户在最后输入一个/。规避方案其实也比较简单，就是上面代码中最后makeMatch处的if else语句，且一定要保证顺序，即最长的要在前面，因为遇到第一个匹配的规则后，后续规则会被忽略。

```go
 http:
  - match:
    - uri:
        prefix: "/echo/"
    - uri:
        prefix: "/echo"
    rewrite:
      uri: "/" 
```

如果顺序颠倒，那么当访问/echo/abc时，会重定向到//abc，返回404错误。

### 总结

至此，已经支持通过统一域名访问，且通过Path把请求转发到不通的服务


