# Kubernetes 启用GPU


## kubernetes设置

1. k8s 1.10之前需要在kube-apiserver、kube-controller-manager、kube-scheduler、kubelet中开启如下feature，如果不是首次部署的话，重启以上所有组件：

     --feature-gates="DevicePlugins=true"

2. 安装 NVIDIA Driver~=361.93

3. 安装nvidia-docker2，由于目前使用的docker版本提示信息里包含自定义字样，只能使用如下方式安装：

     所有安装方式参考[这里](https://github.com/NVIDIA/nvidia-docker)

     ```
   # If you have nvidia-docker 1.0 installed: we need to remove it and all existing GPU containers
      docker volume ls -q -f driver=nvidia-docker | xargs -r -I{} -n1 docker ps -q -a -f volume={} | xargs -r docker rm -f
      sudo yum remove nvidia-docker
      
      # Add the package repositories
      distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
      curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.repo | \
        sudo tee /etc/yum.repos.d/nvidia-container-runtime.repo
      
      # Install the nvidia runtime hook
      sudo yum install -y nvidia-container-runtime-hook
      sudo mkdir -p /usr/libexec/oci/hooks.d
      echo -e '#!/bin/sh\nPATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin" exec nvidia-container-runtime-hook "$@"' | \
        sudo tee /usr/libexec/oci/hooks.d/nvidia
      sudo chmod +x /usr/libexec/oci/hooks.d/nvidia
      
      # Test nvidia-smi with the latest official CUDA image
      # You can't use `--runtime=nvidia` with this setup.
      docker run --rm nvidia/cuda nvidia-smi
   ```

4. 安装nvidia-container-runtime，在上一步中已经安装了对应的yum repo，这里直接执行如下命令即可：

     > 因为使用了上一步的安装方式，所以需要进行这一步的安装，如果是通过yum直接安装的nvidia-docker2，则不需要进行此步。

    ```shell
   # install runtime
   yum install nvidia-container-runtime
   ```

5. update docker daemon，在docker daemon中添加如下配置

     > daemon.json中添加如下配置，可选配置为"default-runtime": "nvidia"，如果不设置默认runtime，则默认使用runc，启动容器是需要指定--runtime=nvidia

     ```shell
   "default-runtime": "nvidia"，
   "runtimes": {
           "nvidia": {
               "path": "/usr/bin/nvidia-container-runtime",
               "runtimeArgs": []
           }
       }
   ```

6. 安装NVIDIA device plugin，插件以daemonset方式部署，如果集群中既有CPU也有GPU节点，可以通过label筛选出GPU节点，无GPU的节点无需部署此程序

     ```shell
   # For Kubernetes v1.8
   kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.8/nvidia-device-plugin.yml
   # For Kubernetes v1.9
   kubectl create -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v1.9/nvidia-device-plugin.yml
   ```

## 测试

   ```shell
   apiVersion: v1
   kind: Pod
   metadata:
     name: cuda-vector-add
   spec:
     restartPolicy: OnFailure
     containers:
       - name: cuda-vector-add
         # https://github.com/kubernetes/kubernetes/blob/v1.7.11/test/images/nvidia-cuda/Dockerfile
         image: "k8s.gcr.io/cuda-vector-add:v0.1"
         resources:
           limits:
             nvidia.com/gpu: 1
     nodeSelector:
       GPU: "true" //测试时自己给对应GPU节点加了GPU=true的label
       
   kubectl create -f test.yaml
   ```

   执行结果：

   ```shell
   [root@vm10-0-13-17 ~]# kubectl get pods -a
   NAME                                   READY     STATUS      RESTARTS   AGE
   cuda-vector-add                        0/1       Completed   0          3h
   nvidia-device-plugin-daemonset-pv6z8   1/1       Running     0          4h
   [root@vm10-0-13-17 ~]# kubectl logs cuda-vector-add
   [Vector addition of 50000 elements]
   Copy input data from the host memory to the CUDA device
   CUDA kernel launch with 196 blocks of 256 threads
   Copy output data from the CUDA device to the host memory
   Test PASSED
   Done
   ```

