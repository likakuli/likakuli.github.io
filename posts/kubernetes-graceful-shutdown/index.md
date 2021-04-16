# Sidecar优雅退出


### 背景

codis集群在接入弹性云测试时发现容器漂移失败，通过集群日志看，提示 调度超时，去界面查看，已经调度成功了（调度成功的标志就是已经有宿主机IP了），状态显示的pending并不一定就是调度失败。但这反应不出来问题出在哪里，接下来就需要到master机器上执行命名，查看日志来分析问题出在哪里

首先，我们要确定是不是调度超时了，可以直接通过 kubectl describe po kirovpre-krds-sf-f3dec-0 来看Pod的创建时间为 20:40:11，调度成功的时间为20:40:12，可以看到调度还是很快的。再看下相关日志，显示20:39:38删除成功，然后等待容器调度，等待30s后发现容器未调度成功，则判定为调度超时。简单梳理下时间线

1. 20:39:38 删除成功 （kube-odin日志）
2. 20:40:11 容器创建成功 （k8s）
3. 20:40:12 容器调度成功  (k8s)

从时间线来看，确实从删除到调度成功耗时超过了30s，但是从容器创建出来到调度成功才1s，大部分耗时是在删除到新创建的阶段。熟悉k8s的同学应该知道，删除Pod的api支持设置删除方式：backgroud、foregroud，区别就是后台删除是异步的，调用api后立马返回，前台删除的话是同步的，会一直等容器真正删除后才返回，默认为backgroud。也就是说20:39:38提示的删除成功只是api调用成功，容器并未真正的删除，容器的删除操作一直在后台执行，直到20:40:11才删除成功，删除后立马创建新的容器。可以通过kubelet的日志证明这一点，下面的日志是经过筛选后的

```shell
I0603 20:39:37.908557    3033 kubelet.go:1913] SyncLoop (DELETE, "api"): "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)"
I0603 20:39:37.908655    3033 kubelet_pods.go:1433] Generating status for "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)"
I0603 20:39:37.908799    3033 kuberuntime_container.go:468] Running preStop hook for container "docker://5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"
I0603 20:39:37.916683    3033 status_manager.go:499] Patch status for pod "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)" with "{}"
I0603 20:39:37.916699    3033 status_manager.go:506] Status for pod "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)" updated successfully: (6, {Phase:Running Conditions:[{Type:Initialized Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2020-05-29 15:07:09 +0800 CST Reason: Message:} {Type:Ready Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2020-05-29 15:07:26 +0800 CST Reason: Message:} {Type:ContainersReady Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2020-05-29 15:07:26 +0800 CST Reason: Message:} {Type:PodScheduled Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2020-05-29 15:07:08 +0800 CST Reason: Message:}] Message: Reason: NominatedNodeName: HostIP:10.89.231.24 PodIP:10.169.92.60 StartTime:2020-05-29 15:07:09 +0800 CST InitContainerStatuses:[] ContainerStatuses:[{Name:agent-kirovpre-krds-ys02 State:{Waiting:nil Running:&ContainerStateRunning{StartedAt:2020-05-29 15:07:18 +0800 CST,} Terminated:nil} LastTerminationState:{Waiting:nil Running:nil Terminated:nil} Ready:true RestartCount:0 Image:registry.kaku.com/kakuonline/kvstore-sidecar-python2-centos7-agent:ea6d410 ImageID:docker-pullable://registry.kaku.com/kakuonline/kvstore-sidecar-python2-centos7-agent@sha256:6d61df206fef0fbfd940e1139d7dff6b0dfaacd847d8d64b2b480cf5afd8a513 ContainerID:docker://5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72} {Name:kirovpre-krds-ys02 State:{Waiting:nil Running:&ContainerStateRunning{StartedAt:2020-05-29 15:07:22 +0800 CST,} Terminated:nil} LastTerminationState:{Waiting:nil Running:nil Terminated:nil} Ready:true RestartCount:0 Image:registry.kaku.com/kakuonline/kvstore-sidecar-krds:stable ImageID:docker-pullable://registry.kaku.com/kakuonline/kvstore-sidecar-krds@sha256:3945a5304ee92b89605831d991874d0b9fdffb89b19fc92d5bbc4969b8a3cf1f ContainerID:docker://2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825}] QOSClass:Burstable})
I0603 20:39:37.916795    3033 kubelet_pods.go:993] Pod "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)" is terminated, but some containers are still running
I0603 20:39:40.975805    3033 kuberuntime_container.go:485] preStop hook for container {"docker" "5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"} completed
I0603 20:39:40.975845    3033 kuberuntime_container.go:563] Killing container "docker://2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825" with 5 second grace period
I0603 20:39:40.975856    3033 kuberuntime_container.go:468] Running preStop hook for container "docker://2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825"
I0603 20:39:41.402077    3033 kubelet_pods.go:993] Pod "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)" is terminated, but some containers are still running
I0603 20:39:44.044590    3033 kuberuntime_container.go:485] preStop hook for container {"docker" "2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825"} completed
I0603 20:39:44.044634    3033 kuberuntime_container.go:580] Killing container {"docker" "2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825"}, but using 2 second grace period override
I0603 20:39:46.217074    3033 kuberuntime_container.go:587] Container "docker://2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825" exited normally
I0603 20:39:46.217139    3033 kuberuntime_container.go:563] Killing container "docker://5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72" with 5 second grace period
I0603 20:39:46.217149    3033 kuberuntime_container.go:468] Running preStop hook for container "docker://5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"
I0603 20:39:46.217139    3033 event.go:221] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"kirovpre-krds-sf-f3dec-0", UID:"01473fb7-a17b-11ea-8d10-c88d83d31d55", APIVersion:"v1", ResourceVersion:"8136942320", FieldPath:"spec.containers{kirovpre-krds-ys02}"}): type: 'Normal' reason: 'Killing' Killing container with id docker://kirovpre-krds-ys02:Need to kill Pod
I0603 20:39:46.882663    3033 kubelet.go:1942] SyncLoop (PLEG): "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)", event: &pleg.PodLifecycleEvent{ID:"01473fb7-a17b-11ea-8d10-c88d83d31d55", Type:"ContainerDied", Data:"2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825"}
I0603 20:39:46.882689    3033 kubelet_pods.go:1433] Generating status for "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)"
I0603 20:39:46.882719    3033 limit.go:46] restartCountLimit: 1, restartCount: 0
I0603 20:39:49.282611    3033 kuberuntime_container.go:485] preStop hook for container {"docker" "5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"} completed
I0603 20:39:49.282673    3033 kuberuntime_container.go:580] Killing container {"docker" "5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"}, but using -3 second grace period override
I0603 20:39:51.402124    3033 kubelet_pods.go:993] Pod "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)" is terminated, but some containers are still running
I0603 20:39:59.515700    3033 kuberuntime_container.go:587] Container "docker://5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72" exited normally
I0603 20:39:59.515776    3033 event.go:221] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"kirovpre-krds-sf-f3dec-0", UID:"01473fb7-a17b-11ea-8d10-c88d83d31d55", APIVersion:"v1", ResourceVersion:"8136942320", FieldPath:"spec.containers{agent-kirovpre-krds-ys02}"}): type: 'Normal' reason: 'Killing' Killing container with id docker://agent-kirovpre-krds-ys02
I0603 20:39:59.516657    3033 plugins.go:391] Calling network plugin cni to tear down pod "kirovpre-krds-sf-f3dec-0_default"
 
 
...
 
 
I0603 20:40:11.516121    3033 kubelet.go:1913] SyncLoop (DELETE, "api"): "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)"
I0603 20:40:11.518521    3033 status_manager.go:519] Pod "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)" fully terminated and removed from etcd
I0603 20:40:11.519489    3033 kubelet.go:1907] SyncLoop (REMOVE, "api"): "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)"
I0603 20:40:11.519514    3033 kubelet.go:2109] Failed to delete pod "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)", err: pod not found
```

从日志里可以看出从kubelet watch到要删除Pod，到Pod真正删除的整个执行过程。其中SyncLoop (DELETE, "api") 代表watch到了删除Pod的api请求，直到最后Pod "kirovpre-krds-sf-f3dec-0_default(01473fb7-a17b-11ea-8d10-c88d83d31d55)" fully terminated and removed from etcd，至此Pod真正的从etcd中消失。正好符合之前的时间点，即39:38 ~ 40:11是在执行Pod的删除操作。

至此，我们可以得出结论，并不是调度慢导致的超时，是删除Pod慢导致的，我们可以看到kube-odin task设置的调度耗时时长不合理，没有考虑到容器删除的时间，这块的修复已经通知相关同事，我们更关注的是为什么删除用了这么长时间

回头再看下Pod中配置的优雅退出时间和容器退出前的preStop配置

1. Pod的优雅推出时间为5s
2. 业务容器配置的preStop为sleep 3s
3. sidecar容器配置的preStop为sleep 3s

根据我们之前的理解，最长5s 容器就会被（强制）删除，但是这次删除耗时为22s (37 ~ 59)，远大于5s。问题出在哪里呢，经过对比代码和日志后发现

1. 首先，此Pod包含sidecar，容器按序退出，先退出sidecar，再退出业务容器
2. 先并行执行所有sidecar的preStop，sleep 3s
3. 然后并行停止业务容器，先执行preStop，sleep 3s，然后在 max(5s-3s, 2s) = 2s 内（强制）删除容器 （默认最小时间为2s，即至少给容器2s的时间用来优雅退出）
4. 最后并行停止sidecar，先执行preStop，sleep 3s，然后在 5 - (3 + 3 + 2) = -3s 内删除容器

上述步骤可以和日志一一对应起来，如下：

```go
# 对应步骤2，从37到40，正好sleep了3s
I0603 20:39:37.908799    3033 kuberuntime_container.go:468] Running preStop hook for container "docker://5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"
I0603 20:39:40.975805    3033 kuberuntime_container.go:485] preStop hook for container {"docker" "5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"} completed
 
 
 
 
# 对应步骤3，从40~44执行sleep 3，44~46执行真正删除容器的操作
I0603 20:39:40.975856    3033 kuberuntime_container.go:468] Running preStop hook for container "docker://2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825"
I0603 20:39:44.044590    3033 kuberuntime_container.go:485] preStop hook for container {"docker" "2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825"} completed
I0603 20:39:44.044634    3033 kuberuntime_container.go:580] Killing container {"docker" "2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825"}, but using 2 second grace period override
I0603 20:39:46.217074    3033 kuberuntime_container.go:587] Container "docker://2e2354889588dc7483d2bb9be27a5253f292374c8e179b12367e0deea8b2d825" exited normally
 
 
 
 
# 对应步骤4，从46~49执行sleep 3，49~59执行真正删除容器的操作
I0603 20:39:46.217149    3033 kuberuntime_container.go:468] Running preStop hook for container "docker://5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"
I0603 20:39:49.282611    3033 kuberuntime_container.go:485] preStop hook for container {"docker" "5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"} completed
I0603 20:39:49.282673    3033 kuberuntime_container.go:580] Killing container {"docker" "5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72"}, but using -3 second grace period override
I0603 20:39:59.515700    3033 kuberuntime_container.go:587] Container "docker://5fe57cf36af267adae571272f234762ad8741922e24074182ff25301e953ec72" exited normally

```


从上面的执行过程可以看到两个问题：

1. sidecar容器多等待了一遍，其实只需要在所有容器删除之前等待一遍就够了
2. 容器删除的时候，剩余可用时长计算有问题，例如最后的-3，当调用docker stop container的api时会把-3作为timeout传过去，而docker api收到负数时会一直等待容器退出，而不会强制删除，这就是为什么最后的sidecar退出用了10s之久

> sidecar的功能当前还是在pull request中，没有合入主干，主干中的代码虽然有sidecar的能力，但是没有区分生命周期，即无法控制sidecar和业务容器启停的顺序，对于kubelet来说两者地位一样，pull reqeust为如下https://github.com/kubernetes/kubernetes/pull/80744，从2019年7月30号提pr，到现在仍未合入主干。里面有很多评论，其中也包括我提的上述两点问题。

### 总结

明白了问题出在哪里，修改其实就很简单了，这里不再多说如何修改。

此次问题排查使我深刻的认识到了一点：一些看起来很容易理解的东西可能正是我们思维定式的误区，越简单的东西我们往往越容易先入为主，不再进行深入思考。而很多问题往往就是这些不起眼的细节日积月累导致的，我们要尽可能的对我们所用到的东西有深刻全面的理解，降低出问题的概率，毕竟线上无小事。有精力的话还是要看看源码，比如 max（graceperiod - cost, 2）,意思就是会为容器至少设置2s的时间，超过后才能会进行强删；再比如外面不传graceperiod时graceperiod用pod的terminationGracePeriod，传的话就是传进来的参数，这就可能导致出现负数的情况，导致不会强删，terminationGracePeriod失效等。

 

