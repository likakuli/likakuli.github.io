# netns泄露


### 1. 揭开面纱

周一，接到RD反馈线上容器网络访问存在异常，具体线上描述如下：

- 上游服务driver-api所有容器访问下游服务duse-api某一容器TCP【telnet测试】连接不通，访问其余下游容器均正常
- 上游服务容器测试下游容器IP连通性【ping测试】正常

从以上两点现象可以得出一个结论：

- 容器的网络设备存在，IP地址连通，但是容器服务进程未启动，端口未启动

但是，当我们和业务RD确认之后，发现业务容器状态正常，业务进程也正运行着。嗯，问题不简单。

此外，飞哥这边排查还有一个结论：

- arp反向解析duse-api特殊容器IP时，不返回MAC地址信息
- 当telnet失败后，立即执行arp，会返回MAC地址信息

当我们拿着arp解析的MAC地址与容器当前的MAC地址作比较时，发现MAC地址不一致。唔，基本上确定问题所在了，net ns泄漏了。执行如下命令验证：

```shell
sudo ip netns ls | while read ns; do sudo ip netns exec $ns ip addr; done | grep inet | grep -v 127 | awk '{print $2}' | sort | uniq -c
```

确实发现该容器对应的IP出现了两次，该容器IP对应了两个网络命名空间，也即该容器的网络命名空间出现了泄漏。

### 2. 误入迷障

当确定了问题所在之后，我们立马调转排查方向，重新投入到net ns泄漏的排查事业当中。

既然net ns出现了泄漏，我们只需要排查被泄露的net ns的成因即可。在具体定位之前，首先补充一个背景：

- ip netns 命令默认扫描 /var/run/netns 目录，从该目录下的文件读取net ns的信息
- 默认情况下，kubelet调用docker创建容器时，docker会将net ns文件隐藏，如果不做特殊处理，我们执行 ip netns 命令将看不到任何数据
- 当前弹性云为了方便排查问题，做了一个特殊处理，将容器的网络命名空间mount到 /var/run/netns 目录 【**注意，这里有个大坑**】

有了弹性云当前的特殊处理，我们就可以知道所有net ns的创建时间，也即 /var/run/netns 目录下对应文件的创建时间。

我们查看该泄漏ns文件的创建时间为2020-04-17 11:34:07，排查范围进一步缩小，只需从该时间点附近排查即可。

接下来，我们分析了该附近时间段，容器究竟遭遇了什么：

- 2020-04-17 11:33:26 用户执行发布更新操作
- 2020-04-17 11:34:24 平台显示容器已启动
- 2020-04-17 11:34:28 平台显示容器启动脚本执行失败
- 2020-04-17 11:36:22 用户重新部署该容器
- 2020-04-17 11:36:31 平台显示容器已删除成功

既然是容器网络命名空间泄漏，则说明再删除容器的时候，没有执行ns的清理操作。【**注：这里由于基础知识不足，导致问题****排查绕了地球一圈**】

我们梳理kubelet在该时间段对该容器的清理日志，核心相关日志展示如下：

```shell
I0417 11:36:30.974674   37736 kubelet_pods.go:1180] Killing unwanted pod "duse-api-sf-xxxxx-0"
I0417 11:36:30.976803   37736 plugins.go:391] Calling network plugin cni to tear down pod "duse-api-sf-80819-0_default"
I0417 11:36:30.983499   37736 kubelet_pods.go:1780] Orphaned pod "4ae28778-805c-11ea-a54c-b4055d1e6372" found, removing pod cgroups
I0417 11:36:30.986360   37736 pod_container_manager_linux.go:167] Attempt to kill process with pid: 48892
I0417 11:36:30.986382   37736 pod_container_manager_linux.go:174] successfully killed all unwanted processes.
```

简单描述流程：

- I0417 11:36:30.974674 根据删除容器执行，执行杀死Pod操作
- I0417 11:36:30.976803 调用cni插件清理网络命名空间
- I0417 11:36:30.983499 常驻协程检测到Pod已终止运行，开始执行清理操作，包括清理目录、cgroup
- I0417 11:36:30.986360 清理cgroup时杀死容器中还未退出的进程
- I0417 11:36:30.986382 显示所有容器进程都已被杀死

这里提示一点：正常情况下，容器退出时，容器内所有进程都已退出。而上面之所以出现清理cgroup时需要杀死容器内未退出进程，是由于常驻协程的检测机制导致的，常驻协程判定Pod已终止运行的条件是：

```go
// podIsTerminated returns true if pod is in the terminated state ("Failed" or "Succeeded").
func (kl *Kubelet) podIsTerminated(pod *v1.Pod) bool {
   // Check the cached pod status which was set after the last sync.
   status, ok := kl.statusManager.GetPodStatus(pod.UID)
   if !ok {
      // If there is no cached status, use the status from the
      // apiserver. This is useful if kubelet has recently been
      // restarted.
      status = pod.Status
   }
   return status.Phase == v1.PodFailed || status.Phase == v1.PodSucceeded || (pod.DeletionTimestamp != nil && notRunning(status.ContainerStatuses))
}
```

这个容器命中了第三个或条件：容器已被标记删除，并且所有业务容器都不在运行中（业务容器启动失败，根本就没运行起来过），但是Pod的sandbox容器可能仍然处于运行状态。

仅依据上面的kubelet日志，难以发现问题所在。我们接着又分析了cni插件的日志，截取cni在删除该Pod容器网络时的日志如下：

```shell
[pid:98497] 2020/04/17 11:36:30.990707 main.go:89: ===== start cni process =====
[pid:98497] 2020/04/17 11:36:30.990761 main.go:90: os env: [CNI_COMMAND=DEL CNI_CONTAINERID=c2ef79f7596b6b558f0c01c0715bac46714eefd1e9966625a09414c7218e1013 CNI_NETNS=/proc/48892/ns/net CNI_ARGS=IgnoreUnknown=1;K8S_POD_NAMESPACE=default;K8S_POD_NAME=duse-api-sf-xxxxx-0;K8S_POD_INFRA_CONTAINER_ID=c2ef79f7596b6b558f0c01c0715bac46714eefd1e9966625a09414c7218e1013 CNI_IFNAME=eth0 CNI_PATH=/home/kaku/kakucloud/cni-plugins/bin LANG=en_US.UTF-8 PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin KUBE_LOGTOSTDERR=--logtostderr=false KUBE_LOG_LEVEL=--v=3 KUBE_ALLOW_PRIV=--allow-privileged=true KUBE_MASTER=--master=https://10.xxx.xxx.xxx:xxxx KUBELET_ADDRESS=--address=0.0.0.0 KUBELET_HOSTNAME=--hostname_override=10.xxx.xxx.xxx KUBELET_POD_INFRA_CONTAINER= KUBELET_ARGS=--network-plugin=cni --cni-bin-dir=/home/kaku/kakucloud/cni-plugins/bin --cni-conf-dir=/home/kaku/kakucloud/cni-plugins/conf --kubeconfig=/etc/kubernetes/kubeconfig/kubelet.kubeconfig --cert-dir=/etc/kubernetes/ssl --log-dir=/var/log/kubernetes --stderrthreshold=3 --allowed-unsafe-sysctls=net.*,kernel.shm*,kernel.msg*,kernel.sem,fs.mqueue.* --pod-infra-container-image=registry.kaku.com/kakuk8s/pause:3.0 --eviction-hard=  --image-gc-high-threshold=75 --image-gc-low-threshold=65 --feature-gates=KubeletPluginsWatcher=false --restart-count-limit=5 --last-upgrade-time=2019-07-01]
[pid:98497] 2020/04/17 11:36:30.990771 main.go:91: stdin : {"cniVersion":"0.3.0","logDir":"/home/kaku/kakucloud/cni-plugins/acllogs","name":"ddcloudcni","type":"aclCni"}
[pid:98497] 2020/04/17 11:36:30.990790 main.go:181: failed to Statfs "/proc/48892/ns/net": no such file or directory
[pid:98497] 2020/04/17 11:36:30.990814 main.go:94: ===== end cni process =====
```

其中，main.go:181行的错误日志一下就抓住了我们的眼球，结合代码分析下：

```go
func cmdDel(args *skel.CmdArgs) error {
	n, _, err := loadConf(args.StdinData)
	if err != nil {
		return err
	}

	netns, err := ns.GetNS(args.Netns)
	if err != nil {
		log.Println(err)     //// Line 181
		return fmt.Errorf("failed to open netns %q: %v", netns, err)
	}
	defer netns.Close()
	...
}
```

可以看到，cni在调用该插件清理容器网络命名空间时，由于181行的错误，导致cni插件提前退出，并没有执行后面的清理操作。唔，终于找到你，小虫子。

这里，我们先简单总结下问题排查至此，得出的阶段性结论：

- 由于容器启动失败，在删除Pod时，常驻协程定时清理非运行状态Pod的cgroup，杀死了Pod的sandbox容器
- 当删除容器命令触发的cni清理操作执行时，发现sandbox的pause进程已退出，定位不到容器的网络命名空间，因此退出cni的清理操作
- 最终容器网络命名空间泄漏

既然，明确了问题所在，我们就赶紧来定制修复方案吧，甚至与，我们很快就给出了一版修复：

- 保证在Pod的所有容器退出之前，不会执行cgroup清理操作

这样就保证了删除容器命令触发的清理操作能够按照顺序执行：

- 杀死所有业务容器
- 执行cni插件清理工作
- 杀死sandbox容器
- 执行cgroup清理工作

我们风风火火的修复了内部版本之后，还验证了社区新版本代码中这块逻辑仍旧保持原样，就想着给社区送温暖（事实证明是妄想）。我们就去开源版本搭建的集群中，复现这个问题。然后噩梦就来了。。。

相同的Pod配置文件，我们在弹性云内部版本几乎能够百分百复现net ns泄漏的问题，而在开源社区版本中，从未出现过一次net ns泄漏。难不成，搞不好，莫不是说，不是我们定位的这个原因？

### 3. 拨云现月

这个结论对我们来说，不是一个好消息。费力不小，不说南辕北辙，但是确实还未发现问题的根因。

为了进一步缩小问题排查范围，我们找内核组同学请教了一个基础知识：

- 在删除net ns时，如果该ns内仍有网络设备，系统自动先删除网络设备，然后再删除ns

掌握了这个基础知识，我们再来排查。既然原生k8s集群不存在net ns泄漏问题，那问题一定由我们定制的某个模块引起。由于net ns泄漏发生在node上，当前弹性云在node节点上部署的模块包含：

- kubelet
- cni plugins
- other tools

由于kubelet已经被排除嫌疑，那么罪魁祸首基本就是cni插件了。对比原生集群与弹性云线上集群的cni插件，发现一个极有可能会造成net ns泄漏的点：

- 定制的cni插件为了排查问题的方便，将容器的网络命名空间文件绑定挂载到了/var/run/netns 目录下 【参考上面的大坑】

我们赶紧着手验证元凶是否就是它。修改cni插件代码，删除绑定挂载操作，然后在测试环境验证。验证结果符合预期，net ns不在泄漏。至此，真相终于大白于天下了。

### 4. 亡羊补牢

当初为net ns做一个绑定挂载，其目的就是为了方便我们排查问题，使得 ip netns 命令能够访问当前宿主上所有Pod的网络命名空间。

但其实一个简单的软链操作就能够实现这个目标。Pod退出时，如果这个软链文件未被清理，也不会引起net ns的泄漏，同时 ls -la /var/run/netns 命令可以清晰的看到哪些net ns仍有效，哪些已无效。

### 5. 事后诸葛

为什么绑定挂载能够导致net ns泄漏呢？这是由linux 网络命名空间特性决定的：

- 只要该命名空间中仍有一个进程存活，或者存在绑定挂载的情况（可能还存在其他情况），该ns就不会被回收
- 而一旦所有进程都已退出，并且也无特殊状况，linux将自动回收该ns

最后，这个问题本身并不复杂，之所以问题存在如此之久，排查如此曲折，主要暴露了我们的基础知识有所欠缺。

好好学习，天天向上，方是王道！



更多精彩内容可关注微信订阅号：幼儿园小班工程师
