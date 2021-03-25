# Endpoint异常变化


### 背景

> k8s 1.12.4 包含自定义功能

线上集群在**批量原地升级**时出现流量异常问题，大体流程如下：

1. 批量摘流，并等待7秒
2. 批量删除容器
3. watch到Endpoint ready 变化，汇总2s内的变化，摘流或者接流（通用的处理方式，幂等）

原地升级是靠修改image实现的，利用的就是k8s原生的能力。第三步中为了降低对第三方API的访问次数，等待2s，汇总2s内所有变化统一调用一次API来进行摘流或者接流。问题表现为上述过程中容器先摘流，再接流（异常），再摘流，最后再接流，期望的场景是容器摘流，完后等待容器重启，正常之后再接流。

### 分析

近期上线了原地重建的功能，出问题的集群都是使用此功能进行发布更新，所以猜测可能和这个功能有关系。在删除集群或者批量漂移容器时，也涉及对应流程，但是一直没有问题，总的排查方向如下：

1. endpoint 变化机制
2. 为什么批量删除时没有出现问题
3. 原地升级和删除有什么差异

#### Endpoint变化机制

众所周知，k8s针对不同的资源类型会有相应的controller与之对应，控制其及其关联资源的生命周期的变化，Endpoint也不例外，在kube-controller-manager中有endpoint controller，查看其逻辑，主要相关的部分如下所示

```go
func (e *EndpointController) syncService(key string) error {
	...
  
	subsets := []v1.EndpointSubset{}
	var totalReadyEps int = 0
	var totalNotReadyEps int = 0

	for _, pod := range pods {
		if len(pod.Status.PodIP) == 0 {
			glog.V(5).Infof("Failed to find an IP for pod %s/%s", pod.Namespace, pod.Name)
			continue
		}
		if !tolerateUnreadyEndpoints && pod.DeletionTimestamp != nil {
			glog.V(5).Infof("Pod is being deleted %s/%s", pod.Namespace, pod.Name)
			continue
		}

		epa := *podToEndpointAddress(pod)

		hostname := pod.Spec.Hostname
		if len(hostname) > 0 && pod.Spec.Subdomain == service.Name && service.Namespace == pod.Namespace {
			epa.Hostname = hostname
		}

		// Allow headless service not to have ports.
		if len(service.Spec.Ports) == 0 {
			if service.Spec.ClusterIP == api.ClusterIPNone {
				subsets, totalReadyEps, totalNotReadyEps = addEndpointSubset(subsets, pod, epa, nil, tolerateUnreadyEndpoints)
				// No need to repack subsets for headless service without ports.
			}
		} else {
			for i := range service.Spec.Ports {
				servicePort := &service.Spec.Ports[i]

				portName := servicePort.Name
				portProto := servicePort.Protocol
				portNum, err := podutil.FindPort(pod, servicePort)
				if err != nil {
					glog.V(4).Infof("Failed to find port for service %s/%s: %v", service.Namespace, service.Name, err)
					continue
				}

				var readyEps, notReadyEps int
				epp := &v1.EndpointPort{Name: portName, Port: int32(portNum), Protocol: portProto}
				subsets, readyEps, notReadyEps = addEndpointSubset(subsets, pod, epa, epp, tolerateUnreadyEndpoints)
				totalReadyEps = totalReadyEps + readyEps
				totalNotReadyEps = totalNotReadyEps + notReadyEps
			}
		}
	}
  
	...

	glog.V(4).Infof("Update endpoints for %v/%v, ready: %d not ready: %d", service.Namespace, service.Name, totalReadyEps, totalNotReadyEps)
	
  ...
}
```

主要的处理函数为syncService，去掉了一些逻辑，主要的处理逻辑在32行，遍历Pod，查看其PodReady Condition是否为true，true的会会把其IP放入subnet的Addresses结构中，否则放入NotReadyAddresses中。Condition主要是kubelet设置的，在generateAPIPodStatus的时候会进行设置，如下

```go
// generateAPIPodStatus creates the final API pod status for a pod, given the
// internal pod status.
func (kl *Kubelet) generateAPIPodStatus(pod *v1.Pod, podStatus *kubecontainer.PodStatus) v1.PodStatus {
	glog.V(3).Infof("Generating status for %q", format.Pod(pod))

	// check if an internal module has requested the pod is evicted.
	for _, podSyncHandler := range kl.PodSyncHandlers {
		if result := podSyncHandler.ShouldEvict(pod); result.Evict {
			return v1.PodStatus{
				Phase:   v1.PodFailed,
				Reason:  result.Reason,
				Message: result.Message,
			}
		}
	}

	s := kl.convertStatusToAPIStatus(pod, podStatus)

	// Assume info is ready to process
	spec := &pod.Spec
	allStatus := append(append([]v1.ContainerStatus{}, s.ContainerStatuses...), s.InitContainerStatuses...)
	s.Phase = getPhase(spec, allStatus)
	// Check for illegal phase transition
	if pod.Status.Phase == v1.PodFailed || pod.Status.Phase == v1.PodSucceeded {
		// API server shows terminal phase; transitions are not allowed
		if s.Phase != pod.Status.Phase {
			glog.Errorf("Pod attempted illegal phase transition from %s to %s: %v", pod.Status.Phase, s.Phase, s)
			// Force back to phase from the API server
			s.Phase = pod.Status.Phase
		}
	}
	kl.probeManager.UpdatePodStatus(pod.UID, s)
	s.Conditions = append(s.Conditions, status.GeneratePodInitializedCondition(spec, s.InitContainerStatuses, s.Phase))
	s.Conditions = append(s.Conditions, status.GeneratePodReadyCondition(spec, s.Conditions, s.ContainerStatuses, s.Phase))
	s.Conditions = append(s.Conditions, status.GenerateContainersReadyCondition(spec, s.ContainerStatuses, s.Phase))
	// Status manager will take care of the LastTransitionTimestamp, either preserve
	// the timestamp from apiserver, or set a new one. When kubelet sees the pod,
	// `PodScheduled` condition must be true.
	s.Conditions = append(s.Conditions, v1.PodCondition{
		Type:   v1.PodScheduled,
		Status: v1.ConditionTrue,
	})

	if kl.kubeClient != nil {
		hostIP, err := kl.getHostIPAnyWay()
		if err != nil {
			glog.V(4).Infof("Cannot get host IP: %v", err)
		} else {
			s.HostIP = hostIP.String()
			if kubecontainer.IsHostNetworkPod(pod) && s.PodIP == "" {
				s.PodIP = hostIP.String()
			}
		}
	}

	return *s
}
```

代码比较直观，根据实际的PodStatus（从docker中获取的信息）生成新的Status，用来更新Pod Status属性，其中会设置各种Condition，通过GeneratePodReadyCondition实现，里面具体又调用了GenerateContainersReadyCondition生成Pod内各container的ContainerReadyCondition，从而设置PodReadyCondition的值，如下

```go
// GeneratePodReadyCondition returns "Ready" condition of a pod.
// The status of "Ready" condition is "True", if all containers in a pod are ready
// AND all matching conditions specified in the ReadinessGates have status equal to "True".
func GeneratePodReadyCondition(spec *v1.PodSpec, conditions []v1.PodCondition, containerStatuses []v1.ContainerStatus, podPhase v1.PodPhase) v1.PodCondition {
	containersReady := GenerateContainersReadyCondition(spec, containerStatuses, podPhase)
	// If the status of ContainersReady is not True, return the same status, reason and message as ContainersReady.
	if containersReady.Status != v1.ConditionTrue {
		return v1.PodCondition{
			Type:    v1.PodReady,
			Status:  containersReady.Status,
			Reason:  containersReady.Reason,
			Message: containersReady.Message,
		}
	}

	// Evaluate corresponding conditions specified in readiness gate
	// Generate message if any readiness gate is not satisfied.
	unreadyMessages := []string{}
	for _, rg := range spec.ReadinessGates {
		_, c := podutil.GetPodConditionFromList(conditions, rg.ConditionType)
		if c == nil {
			unreadyMessages = append(unreadyMessages, fmt.Sprintf("corresponding condition of pod readiness gate %q does not exist.", string(rg.ConditionType)))
		} else if c.Status != v1.ConditionTrue {
			unreadyMessages = append(unreadyMessages, fmt.Sprintf("the status of pod readiness gate %q is not \"True\", but %v", string(rg.ConditionType), c.Status))
		}
	}

	// Set "Ready" condition to "False" if any readiness gate is not ready.
	if len(unreadyMessages) != 0 {
		unreadyMessage := strings.Join(unreadyMessages, ", ")
		return v1.PodCondition{
			Type:    v1.PodReady,
			Status:  v1.ConditionFalse,
			Reason:  ReadinessGatesNotReady,
			Message: unreadyMessage,
		}
	}

	return v1.PodCondition{
		Type:   v1.PodReady,
		Status: v1.ConditionTrue,
	}
}


// GenerateContainersReadyCondition returns the status of "ContainersReady" condition.
// The status of "ContainersReady" condition is true when all containers are ready.
func GenerateContainersReadyCondition(spec *v1.PodSpec, containerStatuses []v1.ContainerStatus, podPhase v1.PodPhase) v1.PodCondition {
	// Find if all containers are ready or not.
	if containerStatuses == nil {
		return v1.PodCondition{
			Type:   v1.ContainersReady,
			Status: v1.ConditionFalse,
			Reason: UnknownContainerStatuses,
		}
	}
	unknownContainers := []string{}
	unreadyContainers := []string{}
	for _, container := range spec.Containers {
		if containerStatus, ok := podutil.GetContainerStatus(containerStatuses, container.Name); ok {
			if !containerStatus.Ready {
				unreadyContainers = append(unreadyContainers, container.Name)
			}
		} else {
			unknownContainers = append(unknownContainers, container.Name)
		}
	}

	// If all containers are known and succeeded, just return PodCompleted.
	if podPhase == v1.PodSucceeded && len(unknownContainers) == 0 {
		return v1.PodCondition{
			Type:   v1.ContainersReady,
			Status: v1.ConditionFalse,
			Reason: PodCompleted,
		}
	}

	// Generate message for containers in unknown condition.
	unreadyMessages := []string{}
	if len(unknownContainers) > 0 {
		unreadyMessages = append(unreadyMessages, fmt.Sprintf("containers with unknown status: %s", unknownContainers))
	}
	if len(unreadyContainers) > 0 {
		unreadyMessages = append(unreadyMessages, fmt.Sprintf("containers with unready status: %s", unreadyContainers))
	}
	unreadyMessage := strings.Join(unreadyMessages, ", ")
	if unreadyMessage != "" {
		return v1.PodCondition{
			Type:    v1.ContainersReady,
			Status:  v1.ConditionFalse,
			Reason:  ContainersNotReady,
			Message: unreadyMessage,
		}
	}

	return v1.PodCondition{
		Type:   v1.ContainersReady,
		Status: v1.ConditionTrue,
	}
}
```

逻辑比较简单，总结一下就是根据container实际的状态，设置pod的状态，只要有一个container not ready，则pod not ready，从而设置pod的各种condition。

#### 批量删除

删除容器时，其实是为Pod设置了deletionTimestamp属性（update事件），继续返回上面syncService的逻辑，第13行，tolerateUnreadyEndpoints默认为false，删除pod时，pod.DeletionTimestamp不为空，就会命中函数体的逻辑，执行continue，从而不会进行condition的判断。最终的效果就是批量删除时，很快就会收到endpoint的update事件，2s后再次进行摘流操作

#### 原地升级

原地升级是批量变更Pod的Image属性，kubelet watch到Pod变化，经过一起列处理，最终来到syncPod函数，但是第一次到来的时候，容器还是running的，最终设置的pod ready condition为true，然后经过computePodAction发现container的hash变了，需要重启container，最终触发killContainer，中间还涉及到优雅删除等问题，最终的效果就是进行了批量原地升级后，并不会立马上报pod not ready，而是经过了一段时间，又因为endpoint的update事件一次更新一个ip，2s内收到的update事件就可能不全，从而导致出现反复的摘流再接流再摘流，对业务造成影响。

### 修改方案

通过mutatingwebhook实现一个通用的能力，针对endpoint的create和update事件，从配置中心（内部组件）中获取对应的配置，并通过规则引擎（开源版本可参考 https://github.com/antonmedv/expr ），对subnet中的Addresses和NotReadyAddresses做一些修改，这样可以实现无侵入式的修改，也比较灵活，可以对配置进行实时修改等，后续像sidecar这种根据用户需求来设置pod ready condition的情况，也无需修改代码，只需要添加配置即可，而且也可以通过condition看到真实的Container、Pod状态



更多精彩内容可关注微信订阅号：幼儿园小班工程师
