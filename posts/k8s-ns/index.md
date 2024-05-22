# f**king k8s - pod namespace


不定期分享 k8s 里面各种坑，Just 避雷

# 结论

太长不看版：在拦截 pod 创建请求时，在业务逻辑中不要直接依赖 admission request 的 pod namespace 属性。可以使用 adminssion request 的 namespace，这个属性是有值的，从请求的 path 中解析得来的。

不明白为什么要在最后才去设置 namespace，以及为什么同样是内部实现，deployment 生成 replicaset 时会为其设置 namespace 再调用 api 创建，而 replicaset 创建 pod 时则不设置 namespace 直接调用 api 创建，有点精神分裂的嫌疑。

# 由来

kube-apiserver 提供了 admission webhook 可以对请求进行拦截处理，其会把请求对应的资源对象传给第三方的 http 服务，第三方 http 服务获取到资源对象后就可以进行自己的逻辑处理，最终按照约定的格式返回对应的处理结果给 k8s。

之前设计实现了一个通用的 admission webhook 策略引擎 kinitiras，具体的业务逻辑则以 cr 的形式存在，cr 中可以设置 resourceSelector 对传入的资源对象进行筛选过滤。

本次的问题就出在 namespace 上，以 mutatingwebhook 为例，现象是外层 mutatingwebhookconfiguration 的配置中并没有设置 namespaceSelector，进入到 webhook 的请求会比较多，内部每个策略通过自己的 resourceSelectors 筛选出来需要执行自身逻辑的资源对象。也就是说将具体的筛选匹配逻辑从外层的 mutatingwebhookconfiguration 放入了里面具体的业务逻辑里面，这样可以用一个通用的 mutatingwebhookconfiguration 文件配合就可以实现所有请求的处理。

cr 资源部分配置如下

```yaml
apiVersion: policy.kcloudlabs.io/v1alpha1
kind: ClusterOverridePolicy
metadata:
  name: test
spec:
  overrideRules:
  - overriders:
      cue: |-
        object: _ @tag(object)
        
        patches: [
         {
            "op": "add"
              "path": "/metadata/annotations/foo~1bar"
              "value":"true"
          },
        ]
  targetOperations:
    - CREATE
resourceSelectors:
  - apiVersion: v1
    kind: Pod
    namespace: test 
```

意思是拦截所有 test ns 下所有的 pod 创建行为，为其添加 `foo/bar: true` 的 annotation。实际的执行效果是在 test 下通过 deployment 创建 pod 时，并没有命中这个策略，annoation 没有添加上，不符合预期。

deployment 如下

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: test
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

# 原因

上述 deployment 托管的 pod 被传递给 webhook 服务端时 namespace 是空的，而 resourceSelectors 要求资源要在 test namespace 下，所以不会命中这条规则。

## 为什么没有 ns

为什么传给 webhook 的 pod namespace 是空的呢？这就涉及到 kube-controller-manager 了，其定义了一个 interface `PodControlInterface` 并提供了相关的实现，内部的控制器需要操作 pod 时都是通过这个接口进行的。其创建 pod 的相关逻辑如下

```go
func GetPodFromTemplate(template *v1.PodTemplateSpec, parentObject runtime.Object, controllerRef *metav1.OwnerReference) (*v1.Pod, error) {
	desiredLabels := getPodsLabelSet(template)
	desiredFinalizers := getPodsFinalizers(template)
	desiredAnnotations := getPodsAnnotationSet(template)
	accessor, err := meta.Accessor(parentObject)
	if err != nil {
		return nil, fmt.Errorf("parentObject does not have ObjectMeta, %v", err)
	}
	prefix := getPodsPrefix(accessor.GetName())

	pod := &v1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Labels:       desiredLabels,
			Annotations:  desiredAnnotations,
			GenerateName: prefix,
			Finalizers:   desiredFinalizers,
		},
	}
	if controllerRef != nil {
		pod.OwnerReferences = append(pod.OwnerReferences, *controllerRef)
	}
	pod.Spec = *template.Spec.DeepCopy()
	return pod, nil
}
```

很简单，从 PodTemplateSpec 生成一个 pod 对象，里面并没有给 pod 设置 namespace 的属性，也就是说不管在 template 里面有没有设置 namespace，最终其生成的 pod 对象都是不带 namespace 的。这一点可以通过对上面的 deployment 做修改，在 template 中为 pod 设置上 namespace 去验证，仍然不会命中。

但如果直接通过如下的 yaml 去创建一个 pod 的话，就可以命中对应的 cr，也就是说 namespace 是有值的。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: test
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

此时 pod 的创建并不会涉及到 kube-controller-manager，外部传什么就是什么。

## 最终又是在哪里设置的 ns

通过 deployment 控制 pod 的情况下，pod 在传递给 webhook 的时候还没设置 namespace 的值，但是保存到 etcd 之后会有 namespace 的值，那么这个属性是在哪里被赋值的呢？

```go
// BeforeCreate ensures that common operations for all resources are performed on creation. It only returns
// errors that can be converted to api.Status. It invokes PrepareForCreate, then Validate.
// It returns nil if the object should be created.
func BeforeCreate(strategy RESTCreateStrategy, ctx context.Context, obj runtime.Object) error {
	...

	// ensure namespace on the object is correct, or error if a conflicting namespace was set in the object
	requestNamespace, ok := genericapirequest.NamespaceFrom(ctx)
	if !ok {
		return errors.NewInternalError(fmt.Errorf("no namespace information found in request context"))
	}
	if err := EnsureObjectNamespaceMatchesRequestNamespace(ExpectedNamespaceForScope(requestNamespace, strategy.NamespaceScoped()), objectMeta); err != nil {
		return err
	}

	...
}

func EnsureObjectNamespaceMatchesRequestNamespace(requestNamespace string, obj metav1.Object) error {
	objNamespace := obj.GetNamespace()
	switch {
	case objNamespace == requestNamespace:
		// already matches, no-op
		return nil

	case objNamespace == metav1.NamespaceNone:
		// unset, default to request namespace
		obj.SetNamespace(requestNamespace)
		return nil

	case requestNamespace == metav1.NamespaceNone:
		// cluster-scoped, clear namespace
		obj.SetNamespace(metav1.NamespaceNone)
		return nil

	default:
		// mismatch, error
		return errors.NewBadRequest("the namespace of the provided object does not match the namespace sent on the request")
	}
}
```

在 pod 被保存到 etcd 之前会从 ctx 中获取 namespace 的值，ctx value 是在请求到来后通过 `WithRequestInfo` 写到 ctx 中的，其 namespace 就是从 url path 中解析得到的。然后调用 `EnsureObjectNamespaceMatchesRequestNamespace` 为 obj namespace 赋值，最后调用 etcd api 创建对象。

# 其他资源是否有 ns

其他资源在传递给 webhook 时，admission request 中的 object 是否设置了 namespace 属性呢？这个就要具体看了，与代码实现有关，比如 deployment 在创建 replicaset 时就为 rs 设置了 namespace。

# 结论

admission webhook 在业务逻辑中不要直接依赖 admission request 的 pod namespace 属性。可以使用 adminssion request 的 namespace，这个属性是有值的，从请求的 path 中解析得来的。是否有值，既合资源有关，也和创建资源的方式有关。

