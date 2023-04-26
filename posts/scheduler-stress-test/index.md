# 调度器压测工具介绍


# 背景

源于一次线上 P0 故障，一个生产集群被误操作删除（不只是业务被删，是集群也被删了），集群规模较大，在集群恢复后 Pod 进行了重新、调度的过程，整个过程（从开始恢复集群到业务服务就绪）耗时略长，其中涉及到调度环节耗时的计算，由于当时监控服务也部署在集群中，导致故障时的调度器监控数据丢失，最后的最后，又回到了原点：故障驱动，自证清白。于是就有了 scheduler-stress-test 项目，就有了本篇关于此项目的介绍，希望可以帮助到有类似需求（调度器压测）的同志们。

# 实现

## Metrics

能想到的最简单直观的办法，就是通过调度器暴露出来的 metrics 来计算调度性能，调度器指标定义文件：

`k8s.io/kubernetes/pkg/scheduler/metrics/metrics.go`

有如下几个关键指标：

| 指标                                                   | type      | query example                                                |
| ------------------------------------------------------ | --------- | ------------------------------------------------------------ |
| scheduler_e2e_scheduling_duration_seconds_count        | count     | sum(rate(scheduler_e2e_scheduling_duration_seconds_count{job="advanced-scheduler",profile="default-scheduler",result="scheduled"}[5m])) by (instance) |
| scheduler_pending_pods                                 | gauge     | scheduler_pending_pods{queue='active', job="default-scheduler"} |
| scheduler_e2e_scheduling_duration_seconds_bucket       | histogram | histogram_quantile(0.99, sum(rate(scheduler_e2e_scheduling_duration_seconds_bucket{job="default-scheduler"}[5m])) by (le)) |
| scheduler_scheduling_algorithm_duration_seconds_bucket | histogram | histogram_quantile(0.99, sum by(le) (rate(scheduler_scheduling_algorithm_duration_seconds_bucket{job="default-scheduler"}[5m]))) |
| scheduler_binding_duration_seconds_bucket              | histogram | histogram_quantile(0.99, sum by(le) (rate(scheduler_binding_duration_seconds_bucket{job="default-scheduler"}[5m]))) |

从名字可以很清晰的看出来指标点的含义，这里不再赘述。

## Condition

第二种方式是通过获取 Pod `PodScheduled` Condition 信息，通过计算其 `LastTransitionTime` 与 `CreationTimestamp` 时间差作为调度耗时。

## 分析

两种方式都可以得到调度耗时相关性能数据，但有一些差异，具体表现为：

前者的耗时比较精确，是调度器内存中保存的耗时，但缺少每个 Pod 的耗时，暴露的是所有 Pod 耗时分布，而 `histogram` 本身就会存在一定的误差。

后者的耗时则包含更多阶段的耗时：

- `LastTransitionTime` 是调度器发起异步 `Bind` 请求且 `kube-apiserver` 收到请求后在实际保存数据到 Etcd 前设置的；
- `CreationTimestamp` 是 `kube-apiserver` 收到创建请求后在保存到 Etcd 之前设置的；

所以 `LastTransitionTime` - `CreationTimestamp` 的结果会包含 Create 请求写 Etcd 的耗时（网络传输、写磁盘）、调度器 watch 到 Pod 的耗时（网络传输）、调度器请求 apiserver 到 apiserver 收到请求进行绑定的耗时（网络传输）等。由于 `metav1.Time` 结构在传输时采用 RFC3339 进行编码，只能精确到秒，因此会损失部分精度。

综上，无论采用那种方式进行统计，结果都会有一些误差，重要的是要理解误差来源，以及每种统计方式的结果代表的含义。在实际测试时，可以同时使用两种方式。

# 项目介绍

[scheduler-stress-test](https://github.com/k-cloud-labs/scheduler-stress-test) 即通过 `Condition` 方式进行统计，使用方式参考 [README.md](https://github.com/k-cloud-labs/scheduler-stress-test/blob/main/README.md)。

## 环境准备

为了模拟大规模调度场景，您可以使用 kwok 创建所需数量的节点。创建的节点可能处于 `NotReady` 状态。为了能够将这些节点用于调度 `Pod`，必须为待调度的 `Pod` 添加一个 `toleration`，以容忍所有 `NoSchedule` 的污点。

为此，您应该执行以下步骤：

1. 在您的 k8s 集群上安装 `kwok`，请参考 https://kwok.sigs.k8s.io/docs/user/kwok-in-cluster/；

2. 在您的 k8s 集群上创建虚拟节点，可以参考如下命令

   ```bash
   cat << EOF > node.yaml 
   apiVersion: v1
   kind: Node
   metadata:
     annotations:
       node.alpha.kubernetes.io/ttl: "0"
       kwok.x-k8s.io/node: fake
     labels:
       beta.kubernetes.io/arch: amd64
       beta.kubernetes.io/os: linux
       kubernetes.io/arch: amd64
       kubernetes.io/hostname: {NODE_NAME}
       kubernetes.io/os: linux
       kubernetes.io/role: agent
       node-role.kubernetes.io/agent: ""
       type: kwok
     name: {NODE_NAME}
   spec:
     taints: # Avoid scheduling actual running pods to fake Node
       - effect: NoSchedule
         key: kwok.x-k8s.io/node
         value: fake
   status:
     allocatable:
       cpu: "64"
       ephemeral-storage: 1Ti
       hugepages-1Gi: "0"
       hugepages-2Mi: "0"
       memory: 250Gi
       pods: "110"
     capacity:
       cpu: "64"
       ephemeral-storage: 1Ti
       hugepages-1Gi: "0"
       hugepages-2Mi: "0"
       memory: 250Gi
       pods: "128"
     nodeInfo:
       architecture: amd64
       bootID: ""
       containerRuntimeVersion: ""
       kernelVersion: ""
       kubeProxyVersion: fake
       kubeletVersion: fake
       machineID: ""
       operatingSystem: linux
       osImage: ""
       systemUUID: ""
     phase: Running
   EOF
   
   # create nodes as you needed
   for i in {0..99}; do sed "s/{NODE_NAME}/kwok-node-$i/g" node.yaml | kubectl apply -f -; done
   ```

## 压测

下载代码并构建：

```bash
git clone https://github.com/k-cloud-labs/scheduler-stress-test.git 

make build
```

该工具支持两个命令：create 和 wait。

create 命令使用指定的模板文件，在 k8s 集群中以指定的并发级别创建指定数量的 pod。

wait 命令等待所有上述创建的 pod 被调度并连续打印结果。

示例：

```bash
# 创建 1000 个 pod，使用 1000 的并发级别（namespace: scheduler-stress-test）
sst create --kubeconfig=/root/.kube/config --count 1000 --concurrency 1000 --pod-template=pod.yaml

# 等待结果
sst wait --kubeconfig=/root/.kube/config --namespace=scheduler-stress-test
```

上述示例使用项目中的 pod.yaml 作为模板，在 k8s 集群的 scheduler-stress-test 命名空间中创建了 1000 个 pod。然后等待并连续打印结果，您可以根据需要修改 pod.yaml 文件。



Enjoy it!!!

