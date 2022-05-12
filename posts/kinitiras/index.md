# admission webhook 花式玩法 - kinitiras


## 项目由来

使用 kubernetes 的同学可能或多或少会有以下的实际业务或者需求场景：

- 为确保安全性，需要对某些资源进行删除保护，例如不允许删除 namespace、crd 定义等；
- 根据服务画像为不同的服务设置不同的属性，这些属性基本是业务无感的，例如设置不同的超售比、设置不同的 Label 以及调度特性从而实现 io 敏感型与 io 密集型服务的反亲和调度等；
- 针对同一个 workload 生成 per pod per config 的配置，例如为特定的业务容器设置 debug 开关、特权模式、日志路径等；
- 集群里面每引入一个组件，可能就会同时引入一个甚至多个 admission webhook，久而久之，集群里面会存在众多的 admission webhook，实现方式、关注的资源都不同，管理起来比较复杂；
- 虽然使用一些工具例如 kube-builder、controller-runtime 等可以快速的创建 admission webhook 的框架，但开发整个功能也需要一定的开发工作量，往往需要开发的业务逻辑比较简单，基本是根据一些规则进行一些决策；
- admission webhook 里的业务逻辑的改动需要升级 wehook 来实现，变更就有可能引入线上稳定性风险；

可以大致归为三类：集群资源管理、admission webhok 自身管理、业务资源定制。基于以上的需求场景，一个通用的可编程的 webhook 规则引擎的想法诞生了。

## 项目介绍

[Kinitiras](https://github.com/k-cloud-labs/kinitiras) 为希腊语，意思为发动机、引擎，贴合 rule engine 的概念。项目通过抽象出来三种策略来实现集群资源的 mutate 和 validate 的逻辑，支持通过 CUE 配置业务逻辑，从而支持了动态编程能力，可以在不变更程序的前提下通过对策略的操作实现所需的能力。

## 能力介绍

### Mutate

内置了 `OverridePolicy` 和 `ClusterOverridePolicy` 来支持 Mutate 能力，前者是 Namespace 级别生效，即只能修改同 Namespace 下的资源对象，后者是 Cluster 级别生效，可修改资源对象不受 Namespace 限制。策略内置一些筛选条件来对目标资源对象进行筛选，对于同一个资源对象，可以有多条策略与之匹配，其生效顺序针对不同类型策略优先使用 `ClusterOverridePolicy`，同一种类型的策略则按照字母顺序依次生效。

例：

```yaml
kind: ClusterOverridePolicy
apiVersion: policy.kcloudlabs.io/v1alpha1
metadata:
  name: add-anno-cop-cue
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Pod
      labelSelector:
        matchLabels:
          kinitiras.kcloudlabs.io/webhook: enabled
  overrideRules:
    - targetOperations:
        - CREATE
      overriders:
        cue: |-
          object: _ @tag(object)
          patches: [
            if object.metadata.annotations == _|_ {
              {
                op: "add"
                path: "/metadata/annotations"
                value: {}
              }
            },
            {
              op: "add"
              path: "/metadata/annotations/added-by"
              value: "cue"
            }
          ]
---
kind: OverridePolicy
apiVersion: policy.kcloudlabs.io/v1alpha1
metadata:
  name: add-anno-op-plaintext
  namespace: default
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Pod
      labelSelector:
        matchLabels:
          kinitiras.kcloudlabs.io/webhook: enabled
  overrideRules:
    - targetOperations:
        - CREATE
      overriders:
        plaintext:
          - path: /metadata/annotations/added-by
            op: add
            value: op
```

上述例子中定义了两种类型的策略，通过 `resourceSelector` 进行目标资源对象的筛选，对于筛选出来的资源对象，将会继续使用 `overrideRules` 里面定义的规则进行判断，如果规则的 `targetOperations` 中包含本次操作类型，则将使用规则内的 ``overriders` 生成最终对象，对比原始对象和最终对象生成 json-patch 所需的 patches 数组返回给 `kube-apiserver`.  支持两种类的 `overriders`：plaintext、cue，前者适用于一些简单场景，后者适用于需要根据传入数据进行额外逻辑处理才能得到预期结果的场景，能力相对前者会更强。

cue 脚本约定了输入输出参数，必须包含这些参数脚本才能成功执行。输入参数只有一个：object，即要操作的资源对象，输出参数为 patches 数组，定义如下：

```cue
object: _ @tag(object)

patch: {
	op: string
	path: string
	value: string
}
// for mutating result
patches: [...patch] 
```

具体数据结构如下：

```go
// ResourceSelector the resources will be selected.
type ResourceSelector struct {
	// APIVersion represents the API version of the target resources.
	// +required
	APIVersion string `json:"apiVersion"`

	// Kind represents the Kind of the target resources.
	// +required
	Kind string `json:"kind"`

	// Namespace of the target resource.
	// Default is empty, which means inherit from the parent object scope.
	// +optional
	Namespace string `json:"namespace,omitempty"`

	// Name of the target resource.
	// Default is empty, which means selecting all resources.
	// +optional
	Name string `json:"name,omitempty"`

	// A label query over a set of resources.
	// If name is not empty, labelSelector will be ignored.
	// +optional
	LabelSelector *metav1.LabelSelector `json:"labelSelector,omitempty"`
}

// Overriders offers various alternatives to represent the override rules.
//
// If more than one alternatives exist, they will be applied with following order:
// - Cue
// - Plaintext
type Overriders struct {
	// Plaintext represents override rules defined with plaintext overriders.
	// +optional
	Plaintext []PlaintextOverrider `json:"plaintext,omitempty"`

	// Cue represents override rules defined with cue code.
	// +optional
	Cue string `json:"cue,omitempty"`
}
```

上述例子的效果是对于任意 Namespace 的 Pod，只要其携带 `kinitiras.kcloudlabs.io/webhook: enabled` Label，则会修改其 annotation 为 added-by: cue，针对 default 下的 Pod，只要其携带 `kinitiras.kcloudlabs.io/webhook: enabled` Label，则会修改其 annotation 为 added-by: op。如果两个策略同时 apply 到集群中，同时在 default 下创建一个 Pod 并携带上述 Label，则最终创建的 Pod annotation 将会是 added-by: op，因为 `OverridePolicy` 后执行，覆盖了 `ClusterOverridePolicy` 的修改。

### Validate

内置了 `ClusterValidatePolicy`， 集群级别生效。针对同一个资源对象，同样可能存在多条匹配的策略，顺序在这里不重要，需要全部策略都通过之后才算整体校验通过，任意一个策略校验失败后将会拒绝请求。

例：

```yaml
apiVersion: policy.kcloudlabs.io/v1alpha1
kind: ClusterValidatePolicy
metadata:
  name: test-delete-ns
spec:
  validateRules:
    - cue: |-
        object: _ @tag(object)
        reject: object.metadata.labels != null && object.metadata.labels["kinitiras.kcloudlabs.io/webhook"] == "enabled"
        validate: {
          if reject{
                  reason: "operation rejected"
          }
          if !reject{
                  reason: ""
          }
          valid: !reject
        }
      targetOperations:
        - DELETE
  resourceSelectors:
    - apiVersion: v1
      kind: Namespace
```

通过 `resourceSelector` 进行目标资源对象的筛选，对于筛选出来的资源对象，将会继续使用 `validateRules` 里面定义的规则进行判断，如果规则的 `targetOperations` 中包含本次操作类型，则将使用 `cue` 脚本进行校验，检验结果返回给 `kube-apiserver`。与 Mutate 不同的是，Validate 只支持 cue 脚本校验。cue 脚本约定了输入输出参数，必须包含这些参数脚本才能成功执行。输入参数有两个：object、oldObject，其中后者只有在校验 `UPDATE` 操作时才需要，输出参数为 validate 结果，定义如下：

```cue
object: _ @tag(object)
oldObject: _ @tag(oldObject)

// for validating result
validate: { 
	reason?: string
	valid: bool
}
```

上述例子的效果为阻止删除带有 `kinitiras.kcloudlabs.io/webhook: enabled` 标签的 namespace。

## 部署

### 部署 CRD 资源

```shell
kubectl apply -f https://raw.githubusercontent.com/k-cloud-labs/pkg/main/charts/_crds/bases/policy.kcloudlabs.io_overridepolicies.yaml
kubectl apply -f https://raw.githubusercontent.com/k-cloud-labs/pkg/main/charts/_crds/bases/policy.kcloudlabs.io_clusteroverridepolicies.yaml
kubectl apply -f https://raw.githubusercontent.com/k-cloud-labs/pkg/main/charts/_crds/bases/policy.kcloudlabs.io_clustervalidatepolicies.yaml
```

### 部署应用程序

在部署之前，需要先根据实际需求修改 `webhook-configuration.yaml` 文件，尽量缩小目标资源对象的范围，减少不必要的请求。调整完所有 yaml 文件之后，只需要执行 apply 即可，如下

```
kubectl apply -f deploy/
```

### 例子

在 examples 文件夹下内置了上述出现过的三个策略，可以 apply 到集群进行尝试。

## 小结

上面对项目由来，能力及使用方式进行了简单介绍，核心还是利用了 admission webhook 来实现。可能有的小伙伴对 admission webhook 的稳定性、性能比较谨慎，鉴于此，这里提供了另外一个项目 [pidalio](https://github.com/k-cloud-labs/pidalio)，通过扩展 `client-go Transport` 来实现，在客户端生效，使用简单，代码层面只需要引用对用的包，并为 `rest.Config` 设置自定义的 `Transport` 即可。如下

```go
import(
	"github.com/k-cloud-labs/pidalio"
)

config.Wrap(pidalio.NewPolicyTransport(config, stopCh).Wrap)
```

但 pidalio 存在一个限制，即只支持 Mutate 操作，且必须使用 client-go 访问 `kube-apiserver`。

