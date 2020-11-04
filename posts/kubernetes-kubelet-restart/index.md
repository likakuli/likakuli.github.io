# Kubelet重启导致容器重启


### 问题描述

在修复cgroup泄漏问题时会现停掉kubelet，待修复完成后启动kubelet组件，重启后收到业务反馈，业务容器重启了。

### 问题排查

这个问题具体原因的排查还是花了一定时间的，下面会列一下大致的排查思路并结合源码（自定义的1.12.4分支）进行分析。

排查过程中涉及到了3个容器，如下

|      | 名称                       | 集群 | 宿主        | 结果   | 重启次数 |
| :--- | :------------------------- | :--- | :---------- | :----- | :------- |
| 1    | auto-srv-cwhttp-sf-30b71-0 | py   | 10.86.98.42 | 重启   | 1        |
| 2    | conf-master-sf-19cf6-0     | us01 | 10.15.29.31 | 重启   | 1        |
| 3    | opensource-sf-dc750-2      | us01 | 10.15.29.31 | 未重启 | 1        |

容器启停相关的组件首先想到的就是kubelet，因此去查看kubelet日志，拿py的举例，重启时间为2020-03-12 10:42:27，所以只需要看这之前的一些日志

这里直接贴出来最后过滤后的日志，省略一些中间过程

```shell
root@ddcloud-underlay-kube-node065.py:~$ grep 0312 /var/log/kubernetes/kubelet.INFO | grep -E "10:42:26|10:42:27" | grep -E "8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56|auto-srv-cwhttp-sf-30b71-0|bfde4b15-cb98-11e8-a3c8-6c92bf85beda" | grep -v -i volume
I0312 10:42:26.101676 2353235 kubelet.go:1887] SyncLoop (ADD, "api"): "lifejuhe-pre-sf-d46e3-0_default(9c048a53-53b2-11ea-b6ae-6c92bf85be08), prime-manager-sf-71a2e-2_default(35bc54e5-4ef1-11ea-b6ae-6c92bf85be08), ai-business-sf-0cac5-19_default(b60c0a18-4d70-11ea-b6ae-6c92bf85be08), disconf-sf-92894-2_default(4ae4dfb8-fee4-11e9-b433-6c92bf85be08), am-base-api-new-sf-de855-4_default(e59f4da9-cd68-11e9-92ca-6c92bf85be08), auto-srv-menu-sf-96f75-1_default(eb6ef2a6-4c7c-11ea-b6ae-6c92bf85be08), pre-000_default(d8f7688a-cc39-11e8-a3c8-6c92bf85beda), auto-wechat-srv-sf-e680a-0_default(9a6e0fc8-051c-11ea-b433-6c92bf85be08), oracle-pre-sf-253a7-0_default(c0cef405-5fa8-11ea-b6ae-6c92bf85be08), auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda), auto-srv-chewu-sf-dc750-0_default(1d9eac73-620a-11ea-b6ae-6c92bf85be08), member-sf-747bb-64_default(4e761a14-634e-11ea-b6ae-6c92bf85be08), ep-arch-tmp-bh-sf-1eaad-4_default(7a2d3f79-001a-11e9-a95b-6c92bf85beda), nsky-htw-h5-extranet-sf-900e8-1_default(10ac4571-08c5-11e9-a95b-6c92bf85beda), fate-saver-pre-sf-5cee9-0_default(39915cd3-f70f-11e9-b433-6c92bf85be08), shortserver-sf-ab776-4_default(d1909587-6362-11ea-b6ae-6c92bf85be08), rainbow-h5-sf-c72e4-3_default(cba38b41-1cbe-11ea-b433-6c92bf85be08), agency-call-service-sf-674c2-29_default(e74635f4-5b46-11ea-b6ae-6c92bf85be08), hnc-ddrccp-sf-db379-1_default(f16fbe5f-9ba4-11e9-a777-6c92bf85be08), log-hnc-sf-6a5be-152_default(a87148b5-c002-11e9-92ca-6c92bf85be08), actinia-service-py-sf-61fb7-0_default(75ef3250-3998-11e9-a95b-6c92bf85beda), its-timing-test-sf-8dcfc-0_default(3ad3b185-d175-11e8-a3c8-6c92bf85beda), de-st-forecastor-sf-5a382-3_default(5c6c02b8-ea6c-11e9-8a9f-6c92bf85be08), transit-compass-sf-69886-3_default(5f28569f-7152-11e9-9a4a-6c92bf85be08), gundam-centos6-002_default(3d075c28-042a-11e9-b650-6c92bf85be08), loan-credit-server-sf-c411b-4_default(6455d8e9-61fe-11ea-b6ae-6c92bf85be08), marketingmodel-service-sf-b1260-16_default(9d570b88-5f31-11e9-9b14-6c92bf85beda), csi-hnc-sf-a8523-16_default(b6078000-5f60-11ea-b6ae-6c92bf85be08), ms-shutdown-datadriver-sf-30b86-4_default(805e9837-f0d7-11e9-a648-6c92bf85be08), abnormal-shutdown-service-sf-342c4-18_default(ccb413e6-d832-11e9-92ca-6c92bf85be08), fate-lancer-sf-49896-2_default(241ad7ce-5c53-11ea-b6ae-6c92bf85be08), jetfire-sf-7dbf2-2_default(13f50c1d-5de8-11ea-b6ae-6c92bf85be08), hnc-pre-v-sf-745f9-0_default(70a98be0-3fbe-11e9-a95b-6c92bf85beda), orderpre-sf-b433d-0_default(03f84b30-1760-11ea-b433-6c92bf85be08), gatewayserver-sf-fc2e7-9_default(310030c4-62b3-11ea-b6ae-6c92bf85be08), base-message-service-sf-1b6f0-2_default(f63261ce-d0aa-11e9-92ca-6c92bf85be08), athena-api-pre-sf-d61bf-0_default(9f7e4183-0089-11ea-b433-6c92bf85be08), soda-f-api-sf-c88f2-2_default(fc9520db-8211-11e9-913a-6c92bf85beda), soda-d-api-py-sf-c6659-5_default(af4a67f9-3216-11ea-b433-6c92bf85be08), rollsroyce-pre-sf-6ed43-0_default(78a053cf-6375-11ea-b6ae-6c92bf85be08), member-sf-747bb-74_default(7323a872-634e-11ea-b6ae-6c92bf85be08), settle-consumer-abtest-hnc-pre-v-sf-b18cf-0_default(d199f994-6363-11ea-b6ae-6c92bf85be08), dpub-vote-pre-sf-bc747-0_default(7388d4f7-1a82-11ea-b433-6c92bf85be08), delta-hub-web-sf-bc67e-2_default(35699c85-620c-11ea-b6ae-6c92bf85be08), drunkeness-model-service-sf-a38a4-43_default(c3cd5daa-5b47-11ea-b6ae-6c92bf85be08), lifeapi-hnc-sf-4a88c-2_default(46c7368a-026c-11e9-a95b-6c92bf85beda), prod-data-service-sf-a1520-0_default(21e423b0-d04f-11e9-92ca-6c92bf85be08), seed-pre-sf-53a95-0_default(c51e0a0f-592a-11ea-b6ae-6c92bf85be08)"
I0312 10:42:26.101950 2353235 predicate.go:80] pod auto-srv-cwhttp-sf-30b71-0 is running on local node, just admit it.
I0312 10:42:26.101969 2353235 prober_manager.go:160] DEFAULT_PROBER: pod auto-srv-cwhttp-sf-30b71-0 is notready state(Running) and has no readiness prober, add a default one.
I0312 10:42:26.102104 2353235 kubelet_pods.go:1424] Generating status for "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)"
I0312 10:42:26.101980 2353235 worker.go:95] INITIAL_STATUS {Running [{Initialized True 0001-01-01 00:00:00 +0000 UTC 2018-10-09 15:55:54 +0800 CST  } {Ready False 0001-01-01 00:00:00 +0000 UTC 2018-12-26 12:14:26 +0800 CST  } {ContainersReady True 0001-01-01 00:00:00 +0000 UTC 2019-04-03 20:02:17 +0800 CST  } {PodScheduled True 0001-01-01 00:00:00 +0000 UTC 2018-10-09 15:55:54 +0800 CST  }]    10.86.98.42 10.161.74.102 2018-10-09 15:55:54 +0800 CST [] [{auto-srv-cwhttp-py {nil &ContainerStateRunning{StartedAt:2018-12-26 12:14:25 +0800 CST,} nil} {nil nil &ContainerStateTerminated{ExitCode:137,Signal:0,Reason:Error,Message:,StartedAt:2018-10-09 15:55:55 +0800 CST,FinishedAt:2018-12-26 11:49:33 +0800 CST,ContainerID:docker://60fcdbc337b3b57fc16c9817de1f5314fefec1f532211972646a0ae7cb64a17d,}} true 1 registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67:20180912 docker-pullable://registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67@sha256:3fb5c481c184f0752db8832f15940c7d306d87a494039a6e667a7561a28fd98b docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56}] Burstable}
I0312 10:42:26.102147 2353235 worker.go:104] LEOPOLD_INIT_PROBER: auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)'s container auto-srv-cwhttp-py has been already running on the host({auto-srv-cwhttp-py {nil &ContainerStateRunning{StartedAt:2018-12-26 12:14:25 +0800 CST,} nil} {nil nil &ContainerStateTerminated{ExitCode:137,Signal:0,Reason:Error,Message:,StartedAt:2018-10-09 15:55:55 +0800 CST,FinishedAt:2018-12-26 11:49:33 +0800 CST,ContainerID:docker://60fcdbc337b3b57fc16c9817de1f5314fefec1f532211972646a0ae7cb64a17d,}} true 1 registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67:20180912 docker-pullable://registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67@sha256:3fb5c481c184f0752db8832f15940c7d306d87a494039a6e667a7561a28fd98b docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56})
I0312 10:42:26.102199 2353235 prober_manager.go:193] add a readiness prober for pod: auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda) - auto-srv-cwhttp-py: &Probe{Handler:Handler{Exec:&ExecAction{Command:[bash -ec [[ -f /etc/container/.ddcloud_initing ]] && { [[ -f /etc/container/.ddcloud_inited ]] || exit 2; }; exit 0;],},HTTPGet:nil,TCPSocket:nil,},InitialDelaySeconds:0,TimeoutSeconds:1,PeriodSeconds:2,SuccessThreshold:1,FailureThreshold:10,}
W0312 10:42:26.102284 2353235 status_manager.go:205] Container readiness changed before pod has synced: "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)" - "docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56"
I0312 10:42:26.119238 2353235 status_manager.go:499] Patch status for pod "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)" with "{\"status\":{\"$setElementOrder/conditions\":[{\"type\":\"Initialized\"},{\"type\":\"Ready\"},{\"type\":\"ContainersReady\"},{\"type\":\"PodScheduled\"}],\"conditions\":[{\"message\":\"containers with unready status: [auto-srv-cwhttp-py]\",\"reason\":\"ContainersNotReady\",\"type\":\"Ready\"},{\"lastTransitionTime\":\"2020-03-12T02:42:26Z\",\"message\":\"containers with unready status: [auto-srv-cwhttp-py]\",\"reason\":\"ContainersNotReady\",\"status\":\"False\",\"type\":\"ContainersReady\"}],\"containerStatuses\":[{\"containerID\":\"docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56\",\"image\":\"registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67:20180912\",\"imageID\":\"docker-pullable://registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67@sha256:3fb5c481c184f0752db8832f15940c7d306d87a494039a6e667a7561a28fd98b\",\"lastState\":{\"terminated\":{\"containerID\":\"docker://60fcdbc337b3b57fc16c9817de1f5314fefec1f532211972646a0ae7cb64a17d\",\"exitCode\":137,\"finishedAt\":\"2018-12-26T03:49:33Z\",\"reason\":\"Error\",\"startedAt\":\"2018-10-09T07:55:55Z\"}},\"name\":\"auto-srv-cwhttp-py\",\"ready\":false,\"restartCount\":1,\"state\":{\"running\":{\"startedAt\":\"2018-12-26T04:14:25Z\"}}}]}}"
I0312 10:42:26.119337 2353235 status_manager.go:506] Status for pod "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)" updated successfully: (1, {Phase:Running Conditions:[{Type:Initialized Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2018-10-09 15:55:54 +0800 CST Reason: Message:} {Type:Ready Status:False LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2018-12-26 12:14:26 +0800 CST Reason:ContainersNotReady Message:containers with unready status: [auto-srv-cwhttp-py]} {Type:ContainersReady Status:False LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2020-03-12 10:42:26 +0800 CST Reason:ContainersNotReady Message:containers with unready status: [auto-srv-cwhttp-py]} {Type:PodScheduled Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2018-10-09 15:55:54 +0800 CST Reason: Message:}] Message: Reason: NominatedNodeName: HostIP:10.86.98.42 PodIP:10.161.74.102 StartTime:2018-10-09 15:55:54 +0800 CST InitContainerStatuses:[] ContainerStatuses:[{Name:auto-srv-cwhttp-py State:{Waiting:nil Running:&ContainerStateRunning{StartedAt:2018-12-26 12:14:25 +0800 CST,} Terminated:nil} LastTerminationState:{Waiting:nil Running:nil Terminated:&ContainerStateTerminated{ExitCode:137,Signal:0,Reason:Error,Message:,StartedAt:2018-10-09 15:55:55 +0800 CST,FinishedAt:2018-12-26 11:49:33 +0800 CST,ContainerID:docker://60fcdbc337b3b57fc16c9817de1f5314fefec1f532211972646a0ae7cb64a17d,}} Ready:false RestartCount:1 Image:registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67:20180912 ImageID:docker-pullable://registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67@sha256:3fb5c481c184f0752db8832f15940c7d306d87a494039a6e667a7561a28fd98b ContainerID:docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56}] QOSClass:Burstable})
I0312 10:42:26.155955 2353235 kubelet.go:1932] SyncLoop (PLEG): "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)", event: &pleg.PodLifecycleEvent{ID:"bfde4b15-cb98-11e8-a3c8-6c92bf85beda", Type:"ContainerStarted", Data:"8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56"}
I0312 10:42:26.155986 2353235 kubelet.go:1932] SyncLoop (PLEG): "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)", event: &pleg.PodLifecycleEvent{ID:"bfde4b15-cb98-11e8-a3c8-6c92bf85beda", Type:"ContainerDied", Data:"60fcdbc337b3b57fc16c9817de1f5314fefec1f532211972646a0ae7cb64a17d"}
I0312 10:42:26.156016 2353235 kubelet_pods.go:1424] Generating status for "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)"
I0312 10:42:26.156102 2353235 kubelet.go:1932] SyncLoop (PLEG): "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)", event: &pleg.PodLifecycleEvent{ID:"bfde4b15-cb98-11e8-a3c8-6c92bf85beda", Type:"ContainerStarted", Data:"e4c7483fc2e3611890959e017d5feb042b57f44fd53f4c470a41da4c45e91731"}
I0312 10:42:26.156132 2353235 kubelet.go:1932] SyncLoop (PLEG): "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)", event: &pleg.PodLifecycleEvent{ID:"bfde4b15-cb98-11e8-a3c8-6c92bf85beda", Type:"ContainerDied", Data:"1d945817797c26545c4b31a441fa5b09d157bff5dd38487d8ff9822b5c022e2f"}
I0312 10:42:26.156156 2353235 kubelet_pods.go:1424] Generating status for "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)"
I0312 10:42:26.402848 2353235 kuberuntime_manager.go:580] computePodActions got {KillPod:false CreateSandbox:false SandboxID:e4c7483fc2e3611890959e017d5feb042b57f44fd53f4c470a41da4c45e91731 Attempt:1 NextInitContainerToStart:nil ContainersToStart:[] ContainersToKill:map[]} for pod "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)"
I0312 10:42:26.403129 2353235 server.go:460] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"auto-srv-cwhttp-sf-30b71-0", UID:"bfde4b15-cb98-11e8-a3c8-6c92bf85beda", APIVersion:"v1", ResourceVersion:"4098877133", FieldPath:""}): type: 'Warning' reason: 'MissingClusterDNS' pod: "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)". kubelet does not have ClusterDNS IP configured and cannot create Pod using "ClusterFirst" policy. Falling back to "Default" policy.
I0312 10:42:26.539178 2353235 prober.go:118] Readiness probe for "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda):auto-srv-cwhttp-py" succeeded
I0312 10:42:27.024882 2353235 kubelet_pods.go:1424] Generating status for "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)"
I0312 10:42:27.032833 2353235 kuberuntime_container.go:561] Killing container "docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56" with 5 second grace period
I0312 10:42:27.032846 2353235 kuberuntime_container.go:466] Running preStop hook for container "docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56"
I0312 10:42:31.908195 2353235 status_manager.go:506] Status for pod "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)" updated successfully: (2, {Phase:Running Conditions:[{Type:Initialized Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2018-10-09 15:55:54 +0800 CST Reason: Message:} {Type:Ready Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2020-03-12 10:42:27 +0800 CST Reason: Message:} {Type:ContainersReady Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2020-03-12 10:42:27 +0800 CST Reason: Message:} {Type:PodScheduled Status:True LastProbeTime:0001-01-01 00:00:00 +0000 UTC LastTransitionTime:2018-10-09 15:55:54 +0800 CST Reason: Message:}] Message:container restart time reaches the limit: 1 Reason:RestartLimit NominatedNodeName: HostIP:10.86.98.42 PodIP:10.161.74.102 StartTime:2018-10-09 15:55:54 +0800 CST InitContainerStatuses:[] ContainerStatuses:[{Name:auto-srv-cwhttp-py State:{Waiting:nil Running:&ContainerStateRunning{StartedAt:2018-12-26 12:14:25 +0800 CST,} Terminated:nil} LastTerminationState:{Waiting:nil Running:nil Terminated:&ContainerStateTerminated{ExitCode:137,Signal:0,Reason:Error,Message:,StartedAt:2018-10-09 15:55:55 +0800 CST,FinishedAt:2018-12-26 11:49:33 +0800 CST,ContainerID:docker://60fcdbc337b3b57fc16c9817de1f5314fefec1f532211972646a0ae7cb64a17d,}} Ready:true RestartCount:1 Image:registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67:20180912 ImageID:docker-pullable://registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67@sha256:3fb5c481c184f0752db8832f15940c7d306d87a494039a6e667a7561a28fd98b ContainerID:docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56}] QOSClass:Burstable})
```

可以看到一个有趣的现象，第19条日志在10:42:26的时候还显示readiness probe探测成功，等到第21条日志10:42:27就去kill容器了，用飞哥的话来说就是 **上一秒还叫人家小甜甜，下一秒就叫人家牛夫人**

```shell
I0312 10:42:26.539178 2353235 prober.go:118] Readiness probe for "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda):auto-srv-cwhttp-py" succeeded
I0312 10:42:27.032833 2353235 kuberuntime_container.go:561] Killing container "docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56" with 5 second grace period
```


这句话是在真正删除容器之前打印出来的，对应代码如下

```go
// killContainer kills a container through the following steps:
// * Run the pre-stop lifecycle hooks (if applicable).
// * Stop the container.
func (m *kubeGenericRuntimeManager) killContainer(pod *v1.Pod, containerID kubecontainer.ContainerID, containerName string, reason string, gracePeriodOverride *int64) error {
   ...

   glog.V(2).Infof("Killing container %q with %d second grace period", containerID.String(), gracePeriod)

   ...
}
```

我们不需要关注他具体的执行过程，只需要按照调用堆栈往上找。killContainer有三处调用，分别对应的情况为

- 有PostStartHook且执行失败时（ContainersToStart包含了所有需要start的容器）
- killPod时
- pod中有需要kill的容器时（ContainersToKill包含了所有需要kill的容器）

从第17条日志中computePodActions的结果可以看到没有需要start和kill的容器，所以调用方为killPod，而killPod也有三处调用，分别为

- updateType为**SyncPodKill**时（容器被驱逐时）
- admit失败或者pod被删除或者pod为failed时
- 容器非首次创建且pod的cgroup不存在时

因为我们没有开启驱逐功能，且此时虽然容器正在运行但是pod的cgroup是存在的，所以只能由中间一条规则触发，也就是必然满足中间的规则，且此时pod没有被删除，也不是failed的状态，所以可以大概判断出来是admit失败导致的容器重启，正好我们自定义了全局的restartcountlimit类型的admit handler，即容器重启次数大于等于1时即可认为admit失败，到这里似乎找到了问题的原因，即**RestartCount大于0的容器会在kubelet停止一段时间重启后导致该容器重启。**然后兴冲冲的跑到py04测试集群测试了一把，果然复现了，然后屁颠屁颠的到群里说发现问题了，但是，飞哥立马质疑说美东的那台宿主上也多个满足条件的容器，但是只有一个重启了。头皮发麻啊，看来还不是问题根本所在，还得继续往前找，为什么同样满足RestartCount大于0，有的admit失败，有的成功，有必要看一下具体实现了，如下

```go
func (l *restartLimitAdmitHandler) Admit(attrs *PodAdmitAttributes) PodAdmitResult {
   status, ok := l.statusManager.GetPodStatus(attrs.Pod.UID)
   if !ok {
      status = attrs.Pod.Status
   }

   isPodReady := func(ps v1.PodStatus) bool {
      for _, cond := range ps.Conditions {
         if cond.Type == v1.PodReady && cond.Status == v1.ConditionTrue {
            return true
         }
      }
      return false
   }
   if status.Phase == v1.PodRunning && isPodReady(status) {
      return PodAdmitResult{Admit: true}
   }

   maxRestartCount := int32(l.config.RestartCountLimit)
   for _, cs := range status.ContainerStatuses {
      if cs.State.Running != nil && cs.Ready {
         return PodAdmitResult{Admit: true}
      }
      if maxRestartCount > 0 && cs.RestartCount >= maxRestartCount {
         return PodAdmitResult{
            Admit:   false,
            Reason:  "RestartLimit",
            Message: fmt.Sprintf("container restart time reaches the limit: %v", maxRestartCount),
         }
      }
   }

   return PodAdmitResult{Admit: true}
}
```

这里需要区分两个概念，Pod Ready和Container Ready。根据问题背景我们知道此时在做cgroup迁移，kubelet已经停了一段时间，且从apiserver的日志中也可以看到node controller检测到NodeNotReady的事件，把上面的所有Pod都设置为Ready:false。即前17行的执行逻辑都是一样的，因为isRodReady函数都是false，再往下看唯一原因就是重启的容器不满足这个条件 

**cs.State.Running != nil && cs.Ready**

但是具体哪个条件不一致导致的，还得继续看日志

```shell
# 有问题的容器的日志
I0312 10:42:26.119238 2353235 status_manager.go:499] Patch status for pod "auto-srv-cwhttp-sf-30b71-0_default(bfde4b15-cb98-11e8-a3c8-6c92bf85beda)" with "{\"status\":{\"$setElementOrder/conditions\":[{\"type\":\"Initialized\"},{\"type\":\"Ready\"},{\"type\":\"ContainersReady\"},{\"type\":\"PodScheduled\"}],\"conditions\":[{\"message\":\"containers with unready status: [auto-srv-cwhttp-py]\",\"reason\":\"ContainersNotReady\",\"type\":\"Ready\"},{\"lastTransitionTime\":\"2020-03-12T02:42:26Z\",\"message\":\"containers with unready status: [auto-srv-cwhttp-py]\",\"reason\":\"ContainersNotReady\",\"status\":\"False\",\"type\":\"ContainersReady\"}],\"containerStatuses\":[{\"containerID\":\"docker://8a61fda8d43e4c28d4092a1bc8e5f372846d955ffffe0353a754c2e42f271b56\",\"image\":\"registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67:20180912\",\"imageID\":\"docker-pullable://registry.kaku.com/kakubuild/auto-srv-cwhttp.hnc-v.chewu-rest.bcar.carplatform.kaku.com.centos67@sha256:3fb5c481c184f0752db8832f15940c7d306d87a494039a6e667a7561a28fd98b\",\"lastState\":{\"terminated\":{\"containerID\":\"docker://60fcdbc337b3b57fc16c9817de1f5314fefec1f532211972646a0ae7cb64a17d\",\"exitCode\":137,\"finishedAt\":\"2018-12-26T03:49:33Z\",\"reason\":\"Error\",\"startedAt\":\"2018-10-09T07:55:55Z\"}},\"name\":\"auto-srv-cwhttp-py\",\"ready\":false,\"restartCount\":1,\"state\":{\"running\":{\"startedAt\":\"2018-12-26T04:14:25Z\"}}}]}}"

# 没问题的容器的日志
I0312 11:12:37.927828 2906779 status_manager.go:499] Patch status for pod "opensource-sf-dc750-2_default(67c1ba01-00f9-11e9-a60d-246e9693c184)" with "{\"status\":{\"$setElementOrder/conditions\":[{\"type\":\"Initialized\"},{\"type\":\"Ready\"},{\"type\":\"ContainersReady\"},{\"type\":\"PodScheduled\"}],\"conditions\":[{\"lastTransitionTime\":\"2020-03-12T03:12:37Z\",\"status\":\"True\",\"type\":\"Ready\"}]}}"
```

对比差异可以看到有问题的容器ready:false，没问题的ready:true，也就对应之前条件里的cs.Ready，因此虽然RestartCount同为1，admit结果却不同。

那ready为啥不一样啊，在哪里赋值的，继续看

```go
func (m *manager) UpdatePodStatus(podUID types.UID, podStatus *v1.PodStatus) {
   for i, c := range podStatus.ContainerStatuses {
      var ready bool
      if c.State.Running == nil {
         ready = false
      } else if result, ok := m.readinessManager.Get(kubecontainer.ParseContainerID(c.ContainerID)); ok {
         ready = result == results.Success
      } else {
         // The check whether there is a probe which hasn't run yet.
         _, exists := m.getWorker(podUID, c.Name, readiness)
         ready = !exists
      }
      podStatus.ContainerStatuses[i].Ready = ready
   }
   // init containers are ready if they have exited with success or if a readiness probe has
   // succeeded.
   for i, c := range podStatus.InitContainerStatuses {
      var ready bool
      if c.State.Terminated != nil && c.State.Terminated.ExitCode == 0 {
         ready = true
      }
      podStatus.InitContainerStatuses[i].Ready = ready
   }
}
```

三处赋值的地方

- c.State.Running为空时（新start的container），此时ready为false，显然并不是这种情况，因为容器之前就已经在运行了
- readiness probe探测结果为success时设置ready:true否则设置false
- 根据readiness探针判断，如果没有此类型探针，则设置ready:true，否则设置为false

这三种情况也都很好理解，针对我们要查的问题，第一种不会触发，第二三种会根据情况触发（所有Pod都默认添加了readiness probe探针，探测容器里的一个指定的文件，dockerinit负责在所有初始化脚本执行成功后创建该文件），也就是先去找探测结果，有的话按探测结果设置，否则按照探针的设置情况来设置，因为我们有默认探针，所以如果走到最后的else里面的话，ready会被设置为false，此时就会触发容器重启。这就需要找不到探测结果才行，直接去找设置探测结果的代码，我们只关注第一次触发添加探测结果的代码，如下

```go
// Creates and starts a new probe worker.
func newWorker(
   m *manager,
   probeType probeType,
   pod *v1.Pod,
   container v1.Container) *worker {

   w := &worker{
      stopCh:       make(chan struct{}, 1), // Buffer so stop() can be non-blocking.
      pod:          pod,
      container:    container,
      probeType:    probeType,
      probeManager: m,
   }
   glog.V(2).Infof("INITIAL_STATUS %v", pod.Status)

   switch probeType {
   case readiness:
      w.spec = container.ReadinessProbe
      w.resultsManager = m.readinessManager
      w.initialValue = results.Failure
      for _, s := range pod.Status.ContainerStatuses {
         if s.Name == w.container.Name && s.Ready {
            glog.V(3).Infof("LEOPOLD_INIT_PROBER: %s's container %s has been already running on the host(%v)",
               format.Pod(w.pod), w.container.Name, s)
            w.containerID = kubecontainer.ParseContainerID(s.ContainerID)
            w.spec.InitialDelaySeconds = 0
            w.initialValue = results.Success
            w.resultsManager.Set(w.containerID, results.Success, w.pod)
            break
         }
      }
   case liveness:
      w.spec = container.LivenessProbe
      w.resultsManager = m.livenessManager
      w.initialValue = results.Success
   }

   w.proberResultsMetricLabels = prometheus.Labels{
      "probe_type":     w.probeType.String(),
      "container_name": w.container.Name,
      "pod_name":       w.pod.Name,
      "namespace":      w.pod.Namespace,
      "pod_uid":        string(w.pod.UID),
   }

   return w
}
```

简单解释一下代码意思，kubelet启动时为每个Pod根据其设置的探针类型创建一个worker，用来进行相关的健康检查，而我们为了实现当kubelet停掉后容器正常运行且kubelet启动后容器不重启的效果，在worker初始化的时候就为每个readiness类型的探针添加了默认的探测结果：success，这样在上面设置ready时，从探测结果获取到success后会把ready设置为true。看到这里你可能会一个疑问

1. 添加默认探测结果的时候也用到了s.Ready，必须为true才能添加默认的探测结果，那为什么又要根据探测结果来设置ready的值？

其实此Ready非彼Ready，添加默认探测结果时用到的Ready为直接从apiserver获取到的etcd中存储的数据，此时确实是Ready:true，因为node controller只设置了Pod Ready:false，并不会去更新Container的属性。而后续我们要设置的Ready才是真整Container的状态，而且此时用到的判断条件都是结合etcd的信息和docker 实际运行容器的状态信息之后做出的判断，用来更新etcd中存储的Pod及Containers的状态，后续会定时把实际运行状态同步到etcd中。

无论有问题的容器还是没有问题的容器，都会进行默认探测结果的设置，都可以看到如下日志

```go
glog.V(2).Infof("INITIAL_STATUS %v", pod.Status)
glog.V(3).Infof("LEOPOLD_INIT_PROBER: %s's container %s has been already running on the host(%v)", format.Pod(w.pod), w.container.Name, s)
```

可以根据这两条日志打印的内容验证上述的逻辑。

那么最后的问题来了，**为什么都设置了默认探测结果，最后的执行结果却不一样**，一个都到了分支2（设置ready = true），一个走到了分支3（设置ready = false）呢？？？

因为添加探测结果和使用探测结果的代码执行是在两个goroutine进行的！！！！！！！不信你看

```go
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
   ...
      // 触发UpdatePod
      kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
      kl.probeManager.AddPod(pod)
}

func (p *podWorkers) UpdatePod(options *UpdatePodOptions) {
   ...
   if podUpdates, exists = p.podUpdates[uid]; !exists {
      ...
      go func() {
         defer runtime.HandleCrash()
         // 触发syncPod, 里面进行ready的赋值
         p.managePodLoop(podUpdates)
      }()
   }
   ...
}


func (m *manager) AddPod(pod *v1.Pod) {
   m.workerLock.Lock()
   defer m.workerLock.Unlock()

   ...
      
      if c.ReadinessProbe != nil {
         key.probeType = readiness
         if _, ok := m.workers[key]; ok {
            glog.Errorf("Readiness probe already exists! %v - %v",
               format.Pod(pod), c.Name)
            return
         }
		 // 添加默认探测成功的结果，分支2使用
         w := newWorker(m, readiness, pod, c)
         w.isDefaultReadinessProber = isDefaultReadinessProber
         // 添加探针，分支3使用
         m.workers[key] = w
         glog.V(3).Infof("add a readiness prober for pod: %v - %v: %v", format.Pod(pod), c.Name, w.spec)
         go w.run()
      }

     ...
}
```

再根据日志打印时间来看

```shell
root@ddcloud-kube-node109.us01:~$ grep "LEOPOLD_INIT_PROBER: opensource-sf-dc750-2" /var/log/kubernetes/kubelet.INFO | head -n 1
I0312 11:12:37.907399 2906779 worker.go:104] LEOPOLD_INIT_PROBER: opensource-sf-dc750-2_default(67c1ba01-00f9-11e9-a60d-246e9693c184)'s container opensource-us01 has been already running on the host({opensource-us01 {nil &ContainerStateRunning{StartedAt:2018-12-26 10:32:29 +0800 CST,} nil} {nil nil &ContainerStateTerminated{ExitCode:255,Signal:0,Reason:Error,Message:,StartedAt:2018-12-16 14:11:20 +0800 CST,FinishedAt:2018-12-26 10:31:31 +0800 CST,ContainerID:docker://0ea96f4f19542f416bf1afba438f7e26d9ef31508829ef5c94d31c184bfb0a6a,}} true 1 registry.kaku.com/kakubuild/opensource.us01-v.opensource.codeinsight.ep.kaku.com.centos67:c2db70ab docker-pullable://registry.kaku.com/kakubuild/opensource.us01-v.opensource.codeinsight.ep.kaku.com.centos67@sha256:ce790b51ef3b617cfd78bf9c1108e66e1c95d87e4590952c5746a5bd4294282d docker://a29de66df4998058df97857223fe2c6c228d4d4ec2f7da7c9da05bcdd7ed214e})


root@ddcloud-kube-node109.us01:~$ grep "Generating status for" /var/log/kubernetes/kubelet.INFO | grep -i opensource | head -n 1
I0312 11:12:37.907439 2906779 kubelet_pods.go:1424] Generating status for "opensource-sf-dc750-2_default(67c1ba01-00f9-11e9-a60d-246e9693c184)"


root@ddcloud-kube-node109.us01:~$ grep "Generating status for" /var/log/kubernetes/kubelet.INFO | grep -i "conf-master" | head -n 1
I0312 11:12:37.907605 2906779 kubelet_pods.go:1424] Generating status for "conf-master-sf-19cf6-0_default(8371b943-0840-11e9-a60d-246e9693c184)"


root@ddcloud-kube-node109.us01:~$ grep "LEOPOLD_INIT_PROBER: conf-master" /var/log/kubernetes/kubelet.INFO | head -n 1
I0312 11:12:37.907629 2906779 worker.go:104] LEOPOLD_INIT_PROBER: conf-master-sf-19cf6-0_default(8371b943-0840-11e9-a60d-246e9693c184)'s container conf-master-us01 has been already running on the host({conf-master-us01 {nil &ContainerStateRunning{StartedAt:2018-12-26 10:32:28 +0800 CST,} nil} {nil nil &ContainerStateTerminated{ExitCode:255,Signal:0,Reason:Error,Message:,StartedAt:2018-12-25 20:28:01 +0800 CST,FinishedAt:2018-12-26 10:31:31 +0800 CST,ContainerID:docker://6e0da293cf06d76938df6b1298595663c964c31c1abddc619a6fc5b5b34dfd04,}} true 1 registry.kaku.com/kakubuild/conf-master.us01-v.cm-server.conf-master.public.engine.kaku.com.centos72:268c0172 docker-pullable://registry.kaku.com/kakubuild/conf-master.us01-v.cm-server.conf-master.public.engine.kaku.com.centos72@sha256:f3b3fbf4a84b85c30f4a8564ce4293bffa440c345c143d9ad3a10904df2b0f7a docker://085089b20cb51d3e3ea6c6bbd04a0e0b467eb9db4e7b6a506bea1151e1d50643})
```

可以看到重启的容器先打印"Generating status for"（此处是给ready赋值之前打印的），再打印的LEOPOLD_INIT_PROBER（添加默认探测结果），也就是说在获取探测结果时其实还没有添加默认探测结果，所以走到了分支3，设置ready:false，而没问题的容器恰好相反，先添加的探测结果，再去使用，走到了分支2。

### 结论

赋值和获取值的操作在两个goroutine，且严重依赖于第一次的赋值操作，所以应该在保证第一次赋值后再进行取值操作才能确保容器不重启，虽然分支3判断有没有默认探针中在设置和获取的时候都加了锁，但还是无法保证代码执行顺序，所以即使走分支3，也有可能会出现设置ready为true的情况（本该设置为false），但是因为这正是我们想要的效果，所以我们是不会觉察到这种情况的问题的，也就是说开源的代码中可能也是存在类似误判的风险的，至少1.12.4版本中是存在的。

