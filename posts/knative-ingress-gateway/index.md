# Knative通过外部域名访问集群内服务


### 背景

> knative 0.14.0
>
> 实际修改可能与贴出来的代码不符，贴出来的代码只是为了方便快速实现功能

最近在搭建公司级的serverless平台，需要用到域名来访问内部服务，采取的是通过PATH来区分不同的服务

### 问题

申请完域名后，分别通过域名和IP:PORT形式访问已部署的helloworld服务

```shell
curl -v -H "Host: api-test.sls.intra.kaku.com" http://api-test.sls.intra.kaku.com/
*   Trying 10.88.128.112...
* TCP_NODELAY set
* Connected to api-test.sls.intra.kaku.com (10.88.128.112) port 80 (#0)
> GET / HTTP/1.1
> Host: api-test.sls.intra.kaku.com
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 426 Upgrade Required
< Date: Thu, 09 Jul 2020 11:59:20 GMT
< Content-Length: 0
< Connection: keep-alive
< server: istio-envoy
<
* Connection #0 to host api-test.sls.intra.kaku.com left intact

# 10.190.16.26 为 knative-ingress-gateway的容器IP
curl -v -H "Host:api-test.sls.intra.kaku.com" http://10.190.16.26/
*   Trying 10.190.16.26...
* TCP_NODELAY set
* Connected to 10.190.16.26 (10.190.16.26) port 80 (#0)
> GET / HTTP/1.1
> Host:api-test.sls.intra.kaku.com
> User-Agent: curl/7.54.0
> Accept: */*
>
< HTTP/1.1 404 Not Found
< date: Thu, 09 Jul 2020 12:03:05 GMT
< server: istio-envoy
< content-length: 0
<
* Connection #0 to host 10.190.16.26 left intact
```

可以看到都无法正常返回，通过域名访问的时候返回了426，通过IP:PORT访问的时候返回了404。

### 排查

#### 426 Upgrade Required

这个问题直接google一搜就出来答案了，参考 [这里](https://github.com/istio/istio/issues/13085)，其实这是envoy的能力，只要在envoy运行的容器中设置ISTIO_META_HTTP10环境变量为"1"问题就解决了，即`**ISTIO_META_HTTP10: '"1"'**`

#### 404 Not Found

这个问题就涉及到VirtualService了，简称vs，在介绍vs之前我们先大致过一下knative创建集群的流程

假设我们通过kubectl操作，此时我们通过`kubectl create -f helloworld.yaml`的方式创建ksvc服务，如果集群各组件正常工作，且ksvc内容正确，那么稍微过一会就可以在集群中看到我们的服务了，我们需要做的仅仅是执行一条命令而已。可以看到**knative封装的太好了，极大的简化了用户操作，对于对集群没有高级需求的用户非常友好，同时也有利于我们快速入门，否则，如果要执行一堆命令的话，就真的可以从入门到放弃了**

但是我们毕竟是管理员，还是要对自己提高要求的，一定要搞清楚里面的原理，各组件之间的交互，否则系统对于我们来说就完全是个黑盒，不出问题还好，出问题就傻眼了。了解源码也是必须的，说到源码，只能感叹knative的源码要比k8s的源码封装的好太多了，其中一个原因也使得益于k8s提供的丰富的扩展机制：crd、operator、informer、webhook等。knative的源码真应该也值得拿出来一起分享，仔细研读。

回到正题，网路路由能力我们选择的是istio，我们大致分两种类型的资源进行介绍，和网络有关的 vs 和网络无关的

**和网络无关的资源创建流程**

`ksvc --> configuration --> revision --> deployment`

**和网络有关的资源创建流程**

`ksvc --> route --> king--> virtualservice`

我们的问题是和网络有关的，所以重点关注下面这个流程，最终对接istio的是vs，于是我们直接去看vs的配置，发现和域名相关的有两个地方，spec.hosts 和 spec.http.match.authority，于是想到的最简单的修改方式就是把我们的域名加入到spec.hosts中，去掉spec.http.match.authority，通过看代码发现这两处并没有可以修改的机制，于是想到利用MutatingWebhook来实现修改

控制这两个属性的地方都在net-istio的controller中，webhook对应的是net-istio的webhook，按照上面的分析，我们需要在webhook中添加对应的代码，主要改动两个地方，如下

```go
// 注册virtualservice类型，表示要对其进行mutate的操作，我们只需要在此注册即可，controller会自动修改对应的MutatingWebhookConfiguration，添加对应的资源和操作
var types = map[schema.GroupVersionKind]resourcesemantics.GenericCRD{
	appsv1.SchemeGroupVersion.WithKind("Deployment"): &defaults.IstioDeployment{},
  v1alpha3.SchemeGroupVersion.WithKind("VirtualService"): &defaults.IstioVirtualService{},
}

// pkg/defaults/virtualservice_default.go，以去掉match.authority举例
package defaults

import (
	"context"

	"istio.io/client-go/pkg/apis/networking/v1alpha3"

	"knative.dev/pkg/apis"
)

type IstioVirtualService struct {
	v1alpha3.VirtualService `json:",inline"`
}

func (vs *IstioVirtualService) Validate(context.Context) *apis.FieldError {
	return nil
}

func (vs *IstioVirtualService) SetDefaults(ctx context.Context) {
	for _, http := range vs.Spec.Http {
		for _, match := range http.Match {
			match.Authority = nil
		}
	}
}
```

可以看到整体修改很少，修改完之后重新编译，制作镜像，修改线上Pod的Image，触发原地重启，然后删除掉原有的vs，新的vs自动生成，查看新的vs，wtf？ 居然和之前一样，没有实现我们的效果，查kube-apiserver日志没有看到在创建vs时调用webhook，查看webhook的日志，也没有发现调用，但是在创建deployment时却会调用，然后查看webhook的配置，发现资源里也已经加上了，查了好久还是没有找到原因，不知道是哪个姿势不对了，由于时间关系暂时换另一种方式实现。

因为vs是由king创建的，所以在创建king的地方修改，这样在king创建vs的时候会自动带上我们自定义的domains，如下

```go
// 通过annotation的方式，把需要添加到hosts中的域名放到annotation中
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
	customHostStr := r.Annotations["serverless.kakuchuxing.com/domains"]
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
```

首先修改ksvc，添加对应的annotaiton，然后继续之前的操作进行编译，打镜像，原地升级，删除vs，新的vs自送生成，此时可以看到已经使我们期望的效果了，然后用域名访问，HelloWorld终于可以正常访问了。

### 总结

问题是解决了，但是为什么通过webhook的方式不生效，现象看起来是没调用webhook，还需要再去看下k8s有关webhook调用的部分的代码，很可能又是一个知识盲区。

knative中很多类型的属性并没有在上层暴露，导致无法直接使用ksvc进行管理，要么改源码，要么自己负责管理原本由ksvc统一管理的组件，虽然更加灵活，但是使用成本也更高，违背ksvc设计的初衷

通过此次问题排查，学习到了knative整个流程、原理，理清了各组件的交互，对后续问题排查有很大的帮助

