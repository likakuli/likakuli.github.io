# Kubernetes 1.9.6 CA IPVS CoreDNS


## 简介



`kubernetes` 系统的各组件需要使用 `TLS` 证书对通信进行加密，本文档使用 `CloudFlare` 的 PKI 工具集 [cfssl](https://github.com/cloudflare/cfssl) 来生成 Certificate Authority (CA) 和其它证书，操作系统CentOS7 amd64；



**集群节点**



- 10.202.43.132	master & etcd

	 10.202.43 133	master & node & etcd

	 10.202.43.134	node & etcd



**认证请求**



- ca-csr.json  

- etcd-csr.json

- admin-csr.json

- kubernetes-csr.json

- kube-proxy-csr.json



**生成的 CA 证书和秘钥文件如下：**



- ca-key.pem

- ca.pem

- kubernetes-key.pem

- kubernetes.pem

- kube-proxy.pem

- kube-proxy-key.pem

- admin.pem

- admin-key.pem



- etcd.pem

- etcd-key.pem



**证书使用情况如下：**



- etcd：使用 ca.pem、etcd-key.pem、etcd.pem；

- kube-apiserver：使用 ca.pem、kubernetes-key.pem、kubernetes.pem；

- kubelet：使用 ca.pem；

- kube-proxy：使用 ca.pem、kube-proxy-key.pem、kube-proxy.pem；

- kubectl：使用 ca.pem、admin-key.pem、admin.pem；



`kube-controller`、`kube-scheduler` 当前需要和 `kube-apiserver` 部署在同一台机器上且使用非安全端口通信，故不需要证书。



## 安装 CFSSL



**这里采用二进制源码包安装，当然也可以使用go命令安装**



```shell

cd /usr/local/bin



wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64

mv cfssl_linux-amd64 cfssl



wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

mv cfssljson_linux-amd64 cfssljson



wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64

mv cfssl-certinfo_linux-amd64 cfssl-certinfo



chmod +x *

```



## 创建 CA (Certificate Authority)



**创建 CA 配置文件**



```shell

mkdir /opt/ssl



cd /opt/ssl



# ca-config.json 文件



vim ca-config.json



{

  "signing": {

    "default": {

      "expiry": "87600h"

    },

    "profiles": {

      "kubernetes": {

        "usages": [

            "signing",

            "key encipherment",

            "server auth",

            "client auth"

        ],

        "expiry": "87600h"

      }

    }

  }

}

```



**字段说明**



- `ca-config.json`：可以定义多个 profiles，分别指定不同的过期时间、使用场景等参数；后续在签名证书时使用某个 profile；

- `signing`：表示该证书可用于签名其它证书，生成的 ca.pem 证书中 `CA=TRUE`；

- `server auth`：表示client可以用该 CA 对server提供的证书进行验证；

- `client auth`：表示server可以用该 CA 对client提供的证书进行验证；



**创建 CA 证书签名请求**



```shell

# ca-csr.json 文件



vim ca-csr.json



{

  "CN": "kubernetes",

  "key": {

    "algo": "rsa",

    "size": 2048

  },

  "names": [

    {

      "C": "CN",

      "ST": "BeiJing",

      "L": "BeiJing",

      "O": "k8s",

      "OU": "System"

    }

  ]

}

```



**字段说明**



- “CN”：`Common Name`，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)，浏览器使用该字段验证网站是否合法；

- “O”：`Organization`，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；



**生成 CA 证书和私钥**



```shell

cfssl gencert -initca ca-csr.json | cfssljson -bare ca



ls

ca.csr  ca-key.pem  ca.pem  ca-csr.json  ca-config.json

```



**分发证书**



```shell

# 创建证书目录

mkdir -p /etc/kubernetes/ssl



# 拷贝所有文件到目录下

cp *.pem /etc/kubernetes/ssl

cp ca.csr /etc/kubernetes/ssl



# 这里要将文件拷贝到所有的k8s 机器上



scp *.pem 10.202.43.133:/etc/kubernetes/ssl/

scp *.csr 10.202.43.133:/etc/kubernetes/ssl/



scp *.pem 10.202.43.134:/etc/kubernetes/ssl/

scp *.csr 10.202.43.134:/etc/kubernetes/ssl/

```



## 创建 etcd 证书



> etcd 证书这里，我做测试时用的etcd地址如下

>

> 10.202.43.132

>

> 10.202.43.133

>

> 10.202.43.134

>

> 实际配置的 IP 还请根据实际情况修改，也可以多预留几个IP，以备后续添加能通过认证，不需要重新签发



```shell

vim etcd-csr.json



{

  "CN": "etcd",

  "hosts": [

    "127.0.0.1",

    "10.202.43.132",   

    "10.202.43.133",

    "10.202.43.134"

  ],

  "key": {

    "algo": "rsa",

    "size": 2048

  },

  "names": [

    {

      "C": "CN",

      "ST": "BeiJing",

      "L": "BeiJing",

      "O": "k8s",

      "OU": "System"

    }

  ]

}



```



**生成 etcd 证书和私钥**



```shell

cfssl gencert -ca=ca.pem \

  -ca-key=ca-key.pem \

  -config=ca-config.json \

  -profile=kubernetes etcd-csr.json | cfssljson -bare etcd



ls etcd*

etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem

```



**拷贝到etcd服务器**



```shell

# etcd-1 

cp etcd*.pem /etc/kubernetes/ssl/



# etcd-2

scp etcd*.pem 10.202.43.133:/etc/kubernetes/ssl/



# etcd-3

scp etcd*.pem 10.202.43.134:/etc/kubernetes/ssl/

```



**修改 etcd 配置**



```shell

# etcd-1



vim /usr/lib/systemd/system/etcd.service



[Unit]

Description=Etcd Server

After=network.target

After=network-online.target

Wants=network-online.target



[Service]

Type=notify

WorkingDirectory=/opt/etcd/

User=etcd

# set GOMAXPROCS to number of processors

ExecStart=/usr/bin/etcd \

  --name=etcd1 \

  --cert-file=/etc/kubernetes/ssl/etcd.pem \

  --key-file=/etc/kubernetes/ssl/etcd-key.pem \

  --peer-cert-file=/etc/kubernetes/ssl/etcd.pem \

  --peer-key-file=/etc/kubernetes/ssl/etcd-key.pem \

  --trusted-ca-file=/etc/kubernetes/ssl/ca.pem \

  --peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem \

  --initial-advertise-peer-urls=https://10.202.43.132:2380 \

  --listen-peer-urls=https://10.202.43.132:2380 \

  --listen-client-urls=https://10.202.43.132:2379,http://127.0.0.1:2379 \

  --advertise-client-urls=https://10.202.43.132:2379 \

  --initial-cluster-token=k8s-etcd-cluster \

  --initial-cluster=etcd1=https://10.202.43.132:2380,etcd2=https://10.202.43.133:2380,etcd3=https://10.202.43.134:2380 \

  --initial-cluster-state=new \

  --data-dir=/opt/etcd/

Restart=on-failure

RestartSec=5

LimitNOFILE=65536



[Install]

WantedBy=multi-user.target

```



另外两个节点上的配置和上面大致相同，只需要把对应IP换成自己的就可以了



**证书相关的配置**



```shell

--cert-file=/etc/kubernetes/ssl/etcd.pem 

--key-file=/etc/kubernetes/ssl/etcd-key.pem 

--peer-cert-file=/etc/kubernetes/ssl/etcd.pem 

--peer-key-file=/etc/kubernetes/ssl/etcd-key.pem 

--trusted-ca-file=/etc/kubernetes/ssl/ca.pem 

--peer-trusted-ca-file=/etc/kubernetes/ssl/ca.pem 

```



**分别启动 etcd 集群**



```shell

systemctl daemon-reload



systemctl start etcd



# 配置开机启动



systemctl enable etcd



# 查看etcd状态



systemctl status etcd

```



**验证 etcd 集群状态**



```shell

etcdctl --endpoints=https://10.202.43.132:2379,https://10.202.43.133:2379,https://10.202.43.134:2379\

        --cert-file=/etc/kubernetes/ssl/etcd.pem \

        --ca-file=/etc/kubernetes/ssl/ca.pem \

        --key-file=/etc/kubernetes/ssl/etcd-key.pem \

        cluster-health

```



## 创建 kubernetes 证书



**创建 admin 证书和私钥**



```shell

# 创建证书签名请求



vim admin-csr.json



{

  "CN": "admin",

  "hosts": [],

  "key": {

    "algo": "rsa",

    "size": 2048

  },

  "names": [

    {

      "C": "CN",

      "ST": "BeiJing",

      "L": "BeiJing",

      "O": "system:masters",

      "OU": "System"

    }

  ]

}



# 生成 admin 证书和私钥



cfssl gencert -ca=ca.pem \

  -ca-key=ca-key.pem \

  -config=ca-config.json \

  -profile=kubernetes admin-csr.json | cfssljson -bare admin



# 查看生成



ls admin*

admin.csr  admin-csr.json  admin-key.pem  admin.pem



cp admin*.pem /etc/kubernetes/ssl/



scp admin*.pem 10.202.43.133:/etc/kubernetes/ssl/

```



- 后续 `kube-apiserver` 使用 `RBAC` 对客户端(如 `kubelet`、`kube-proxy`、`Pod`)请求进行授权；

- `kube-apiserver` 预定义了一些 `RBAC` 使用的 `RoleBindings`，如 `cluster-admin` 将 Group `system:masters` 与 Role `cluster-admin` 绑定，该 Role 授予了调用`kube-apiserver` 的**所有 API**的权限；

- OU 指定该证书的 Group 为 `system:masters`，`kubelet` 使用该证书访问 `kube-apiserver` 时 ，由于证书被 CA 签名，所以认证通过，同时由于证书用户组为经过预授权的 `system:masters`，所以被授予访问所有 API 的权限；



### 配置 kubectl



> 生成证书相关的配置文件存储与 /root/.kube 目录中



```shell

# 配置 kubernetes 集群



kubectl config set-cluster kubernetes \

  --certificate-authority=/etc/kubernetes/ssl/ca.pem \

  --embed-certs=true \

  --server=https://127.0.0.1:6443



# 配置 客户端认证



kubectl config set-credentials admin \

  --client-certificate=/etc/kubernetes/ssl/admin.pem \

  --embed-certs=true \

  --client-key=/etc/kubernetes/ssl/admin-key.pem

  

kubectl config set-context kubernetes \

  --cluster=kubernetes \

  --user=admin



kubectl config use-context kubernetes

```



**创建 kubernetes 证书和私钥**



```shell

# 创建证书签名请求



vim kubernetes-csr.json



{

  "CN": "kubernetes",

  "hosts": [

    "127.0.0.1",

    "10.202.43.132",

    "10.202.43.133",

    "10.254.0.1",

    "kubernetes",

    "kubernetes.default",

    "kubernetes.default.svc",

    "kubernetes.default.svc.cluster",

    "kubernetes.default.svc.cluster.local"

  ],

  "key": {

    "algo": "rsa",

    "size": 2048

  },

  "names": [

    {

      "C": "CN",

      "ST": "BeiJing",

      "L": "BeiJing",

      "O": "k8s",

      "OU": "System"

    }

  ]

}



# 生成证书和私钥



cfssl gencert -ca=ca.pem \

  -ca-key=ca-key.pem \

  -config=ca-config.json \

  -profile=kubernetes kubernetes-csr.json | cfssljson -bare kubernetes



# 查看生成



ls kubernetes*

kubernetes.csr  kubernetes-key.pem  kubernetes.pem  kubernetes-csr.json



# 拷贝到目录

cp kubernetes*.pem /etc/kubernetes/ssl/



scp kubernetes*.pem 10.202.43.133:/etc/kubernetes/ssl/

```



这里 hosts 字段中 三个 IP 分别为 127.0.0.1 本机， 10.202.43.132 和 10.202.43.133 为 Master 的 IP ，多个Master需要写多个，也可以多写几个，方便以后扩展master节点； 10.254.0.1 为 `kue-apiserver` 指定的 `service-cluster-ip-range` 网段的第一个 IP，如 10.254.0.1



### 配置 kube-apiserver



> kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token 一致，如果一致则自动为 kubelet生成证书和秘钥。有关 TLS Bootstrapping  参考 [这里](https://kubernetes.io/docs/admin/kubelet-tls-bootstrapping/)



```shell

# 生成 token



head -c 16 /dev/urandom | od -An -t x | tr -d ' '

f6280a3754345875d392258bd340ef7e



# 创建 token.csv 文件



cd /opt/ssl



vim token.csv



f6280a3754345875d392258bd340ef7e,kubelet-bootstrap,10001,"system:kubelet-bootstrap"



# 拷贝



cp token.csv /etc/kubernetes/



scp token.csv 10.202.43.133:/etc/kubernetes/



# 生成高级审核配置文件



cd /etc/kubernetes



cat >> audit-policy.yaml <<EOF

# Log all requests at the Metadata level.

apiVersion: audit.k8s.io/v1beta1

kind: Policy

rules:

- level: Metadata

EOF



# 拷贝



scp audit-policy.yaml 10.202.43.133:/etc/kubernetes/

```



```shell

vim /usr/lib/systemd/system/kube-apiserver.service



[Unit]

Description=Kubernetes API Server

Documentation=https://github.com/GoogleCloudPlatform/kubernetes

After=network.target



[Service]

User=root

ExecStart=/usr/local/bin/kube-apiserver \

  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \

  --advertise-address=10.202.43.132 \

  --allow-privileged=true \

  --apiserver-count=2 \

  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \

  --audit-log-maxage=30 \

  --audit-log-maxbackup=3 \

  --audit-log-maxsize=100 \

  --audit-log-path=/var/log/kubernetes/audit.log \

  --authorization-mode=Node,RBAC \

  --bind-address=0.0.0.0 \

  --secure-port=6443 \

  --client-ca-file=/etc/kubernetes/ssl/ca.pem \

  --enable-swagger-ui=true \

  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \

  --etcd-certfile=/etc/kubernetes/ssl/etcd.pem \

  --etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \

  --etcd-servers=https://10.202.43.132:2379,https://10.202.43.133:2379,https://10.202.43.134:2379 \

  --event-ttl=1h \

  --kubelet-https=true \

  --insecure-bind-address=127.0.0.1 \

  --insecure-port=8080 \

  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \

  --service-cluster-ip-range=10.254.0.0/16 \

  --service-node-port-range=10000-60000 \

  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \

  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \

  --enable-bootstrap-token-auth \

  --token-auth-file=/etc/kubernetes/token.csv \

  --v=1

Restart=on-failure

RestartSec=5

Type=notify

LimitNOFILE=65536



[Install]

WantedBy=multi-user.target

```



**证书相关配置**



```shell

--admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction    								# 1.8开始需要添加 NodeRestriction

--audit-policy-file=/etc/kubernetes/audit-policy.yaml

--authorization-mode=Node,RBAC  		#1.8 开始需要添加 Node

--client-ca-file=/etc/kubernetes/ssl/ca.pem 

--etcd-cafile=/etc/kubernetes/ssl/ca.pem 

--etcd-certfile=/etc/kubernetes/ssl/etcd.pem 

--etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem 

--kubelet-https=true 

--insecure-bind-address=127.0.0.1    	#不是0.0.0.0 仅供kube-controller-manager 和 kube-scheduler 使用

--insecure-port=8080 

--service-account-key-file=/etc/kubernetes/ssl/ca-key.pem 

--tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem 

--tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem 

--enable-bootstrap-token-auth 

--token-auth-file=/etc/kubernetes/token.csv 

```



### 配置 kube-controller-manager



```shell

# 创建 kube-controller-manager.service 文件



vim /usr/lib/systemd/system/kube-controller-manager.service



[Unit]

Description=Kubernetes Controller Manager

Documentation=https://github.com/GoogleCloudPlatform/kubernetes



[Service]

ExecStart=/usr/local/bin/kube-controller-manager \

  --address=0.0.0.0 \

  --master=http://127.0.0.1:8080 \

  --service-cluster-ip-range=10.254.0.0/16 \

  --cluster-name=kubernetes \

  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \

  --root-ca-file=/etc/kubernetes/ssl/ca.pem \

  --leader-elect=true \

  --v=1

Restart=on-failure

RestartSec=5



[Install]

WantedBy=multi-user.target

```



**证书相关配置**



```shell

--service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem 

--root-ca-file=/etc/kubernetes/ssl/ca.pem 

```



### 配置 kube-scheduler



```shell

# 创建 kube-cheduler.service 文件



vim /usr/lib/systemd/system/kube-scheduler.service



[Unit]

Description=Kubernetes Scheduler

Documentation=https://github.com/GoogleCloudPlatform/kubernetes



[Service]

ExecStart=/usr/local/bin/kube-scheduler \

  --address=0.0.0.0 \

  --master=http://127.0.0.1:8080 \

  --leader-elect=true \

  --v=1

Restart=on-failure

RestartSec=5



[Install]

WantedBy=multi-user.target

```



### 配置 kubelet



> kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper 角色，然后 kubelet 才有权限创建认证请求(certificatesigningrequests)。



```shell

# 先创建认证请求

# user 为 master 中 token.csv 文件里配置的用户

# 只需创建一次就可以



kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

```



**配置 kubelet.kubeconfig**



```shell

# 配置集群，server参数为apiserver地址，可以使用haproxy或nginx来代替直接使用ip,达到k8s多主的目的



kubectl config set-cluster kubernetes \

  --certificate-authority=/etc/kubernetes/ssl/ca.pem \

  --embed-certs=true \

  --server=https://10.202.43.132:6443 \

  --kubeconfig=bootstrap.kubeconfig



# 配置客户端认证



kubectl config set-credentials kubelet-bootstrap \

  --token=f6280a3754345875d392258bd340ef7e \

  --kubeconfig=bootstrap.kubeconfig



# 配置关联



kubectl config set-context default \

  --cluster=kubernetes \

  --user=kubelet-bootstrap \

  --kubeconfig=bootstrap.kubeconfig

  

# 配置默认关联

kubectl config use-context default --kubeconfig=bootstrap.kubeconfig



# 拷贝生成的 bootstrap.kubeconfig 文件



mv bootstrap.kubeconfig /etc/kubernetes/

```



**配置 kubelet.service**



```shell

# 创建 kubelet 目录



mkdir /var/lib/kubelet



vim /usr/lib/systemd/system/kubelet.service



[Unit]

Description=Kubernetes Kubelet

Documentation=https://github.com/GoogleCloudPlatform/kubernetes

After=docker.service

Requires=docker.service



[Service]

WorkingDirectory=/var/lib/kubelet

ExecStart=/usr/local/bin/kubelet \

  --cgroup-driver=cgroupfs \

  --pod-infra-container-image=repository.gridsum.com:8443/kaku/pause-adm64:3.0 \

  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \

  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \

  --cert-dir=/etc/kubernetes/ssl \

  --cluster_dns=10.254.210.250 \

  --cluster_domain=cluster.local. \

  --allow-privileged=true \

  --fail-swap-on=false \

  --serialize-image-pulls=false \

  --logtostderr=true \

  --max-pods=512 \

  --v=1



[Install]

WantedBy=multi-user.target

```



**证书相关配置**



```shell

--experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig 

--kubeconfig=/etc/kubernetes/kubelet.kubeconfig 

--cert-dir=/etc/kubernetes/ssl 

```



**配置 TLS 认证**



```shell

# 查看 csr 的名称



kubectl get csr

NAME                                                   AGE       REQUESTOR           CONDITION

node-csr-***********************************		  1m        kubelet-bootstrap   Pending





# 增加 认证



kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve



# 成功以后会自动生成配置文件与密钥



# 配置文件



ls /etc/kubernetes/kubelet.kubeconfig   

/etc/kubernetes/kubelet.kubeconfig





# 密钥文件  这里注意如果 csr 被删除了，请删除如下文件，并重启 kubelet 服务



ls /etc/kubernetes/ssl/kubelet*

/etc/kubernetes/ssl/kubelet-client.crt  /etc/kubernetes/ssl/kubelet.crt

/etc/kubernetes/ssl/kubelet-client.key  /etc/kubernetes/ssl/kubelet.key

```



### 配置 kube-proxy



**创建 kube-proxy 证书和私钥**



```shell

vim kube-proxy-csr.json



{

  "CN": "system:kube-proxy",

  "hosts": [],

  "key": {

    "algo": "rsa",

    "size": 2048

  },

  "names": [

    {

      "C": "CN",

      "ST": "BeiJing",

      "L": "BeiJing",

      "O": "k8s",

      "OU": "System"

    }

  ]

}



cfssl gencert -ca=ca.pem \

  -ca-key=ca-key.pem \

  -config=ca-config.json \

  -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

  

# 查看生成

ls kube-proxy*

kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem



# 拷贝到目录



cp kube-proxy* /etc/kubernetes/ssl/



scp kube-proxy* 10.202.43.133:/etc/kubernetes/ssl/



scp kube-proxy* 10.202.43.134:/etc/kubernetes/ssl/

```



- CN 指定该证书的 User 为 `system:kube-proxy`；

- `kube-apiserver` 预定义的 RoleBindings `cluster-admin` 将User `system:kube-proxy` 与 Role `system:node-proxier` 绑定，该 Role 授予了调用 `kube-apiserver` Proxy 相关 API 的权限；



**配置 kube-proxy.kubeconfig**



```shell

# 配置集群, server参数为apiserver地址，可以使用haproxy或nginx来代替直接使用ip,达到k8s多主的目的



kubectl config set-cluster kubernetes \

  --certificate-authority=/etc/kubernetes/ssl/ca.pem \

  --embed-certs=true \

  --server=https://10.202.43.132:6443 \

  --kubeconfig=kube-proxy.kubeconfig



# 配置客户端认证



kubectl config set-credentials kube-proxy \

  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \

  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \

  --embed-certs=true \

  --kubeconfig=kube-proxy.kubeconfig

  

# 配置关联



kubectl config set-context default \

  --cluster=kubernetes \

  --user=kube-proxy \

  --kubeconfig=kube-proxy.kubeconfig



# 配置默认关联

kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig



# 拷贝到需要的 node 端里



scp kube-proxy.kubeconfig 10.202.43.133:/etc/kubernetes/



scp kube-proxy.kubeconfig 10.202.43.134:/etc/kubernetes/

```



**配置 kube-proxy.service**



> 1.9 官方 ipvs 已经 beta , 尝试开启 ipvs 测试一下, 官方 –feature-gates=SupportIPVSProxyMode=false 默认是 false 的， 需要设置 –-feature-gates=SupportIPVSProxyMode=true –-masquerade-all。执行 yum install ipvsadm -y 安装 ipvs，ipvs 和 calico 不兼容，原因之一是 calico 必须不能设置 –-masquerade-all



```shell

# 创建 kube-proxy 目录



mkdir -p /var/lib/kube-proxy



vim /usr/lib/systemd/system/kube-proxy.service



[Unit]

Description=Kubernetes Kube-Proxy Server

Documentation=https://github.com/GoogleCloudPlatform/kubernetes

After=network.target



[Service]

WorkingDirectory=/var/lib/kube-proxy

ExecStart=/usr/local/bin/kube-proxy \

  --bind-address=10.202.43.133 \

  --hostname-override=10.202.43.133 \

  --masquerade-all \

  --feature-gates=SupportIPVSProxyMode=true \

  --proxy-mode=ipvs \

  --ipvs-min-sync-period=5s \

  --ipvs-sync-period=5s \

  --ipvs-scheduler=rr \

  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \

  --logtostderr=true \

  --v=1

Restart=on-failure

RestartSec=5

LimitNOFILE=65536



[Install]

WantedBy=multi-user.target



# 配置说明

# --bind-address 和 --hostname-override 按需设置， 其中 --bind-address 为 kube-proxy 所在节点的 IP

```



**证书相关配置**



```shell

--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

```



### 配置 CoreDNS



**配置 coredns.yaml**



```shell

# 下载配置文件



wget https://raw.githubusercontent.com/coredns/deployment/master/kubernetes/coredns.yaml.sed



mv coredns.yaml.sed coredns.yaml



# vim coredns.yaml



...

data:

  Corefile: |

    .:53 {

        errors

        health

        kubernetes cluster.local 10.254.0.0/16 {

          pods insecure

          upstream /etc/resolv.conf

          fallthrough in-addr.arpa ip6.arpa

        }

...        

        image: repository.gridsum.com:8443/library/coredns

...        

  clusterIP: 10.254.210.250



# 创建 coredns



kubectl apply -f coredns.yaml 

```



## dashboard



> 官方 [dashboard](https://github.com/kubernetes/dashboard)



dashboard都从1.7之后只支持https访问，按照官网使用kubectl proxy启动本地代理来访问。此处必须为本地代理，即必须通过[`http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/`](http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/)访问，如果要在windows上访问的话，可以自己编译一个kubectl对应的windows客户端，通过令牌登陆的时候，能看到的内容与token的权限有关。



## apiserver访问



**客户端在集群外**



```shell

# 在任一 master 节点执行瑞安命令便可在集群外通过http://masterip:8001访问api server



kubectl proxy --address 0.0.0.0 --accept-hosts '.*' --port=8001

```



**集群内访问**



```shell

# kubectl get secret

NAME                  TYPE                                  DATA      AGE

default-token-z6lq7   kubernetes.io/service-account-token   3         15d



# kubectl describe secret default-token-z6lq7

Name:         default-token-z6lq7

Namespace:    default

Labels:       <none>

Annotations:  kubernetes.io/service-account.name=default

              kubernetes.io/service-account.uid=c246a42f-3d73-11e8-aa36-00155d010a3a



Type:  kubernetes.io/service-account-token



Data

====

ca.crt:     1310 bytes

namespace:  7 bytes

token:      eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tejZscTciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImMyNDZhNDJmLTNkNzMtMTFlOC1hYTM2LTAwMTU1ZDAxMGEzYSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.Qfi9881mVeX8Z4ophf6_l5e5jGcILFobS5mKTQgdoz6NF0rJ4buRASmQiqu4N5ErolOstSIJhhK1yVzHbkBYsYReip6ffTnOwF2cWU5EJAhP7_o2zGWK5b11amlp5qLU0rWucfYe34ZfGxAcxoekmgKwJ6Hu58JlgD3ae5lu-_J6yVT_O-klC6FUXCY-r3FbtwwYz7WbrSxhuu5nCbegmEy5gPy9aEeVZcz6v5ZIyTU62mvbO_M1xYQfPyaHPWgjbkh9H540j2LUa7Y7RQw_LLEp_NbpzVfN58sFLY8cndzUfr5v_KjtFXyPVqmUX0qxoIRWL2opB7Qr2lL7qyTg5w



#上述secret会挂载到所有的pod上，容器内对应路径为/var/run/secrets/kubernetes.io/serviceaccount

#集群内的Pod通过在http请求的header中添加Authorization: Bearer ${KUBE_BEARER_TOKEN} 进行访问



# kubectl exec -it crawler-bgapi-new-9bjc7 sh 

# cd /var/run/secrets/kubernetes.io/serviceaccount

# ls

ca.crt	namespace  token

# cat token

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZWZhdWx0Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6ImRlZmF1bHQtdG9rZW4tejZscTciLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGVmYXVsdCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImMyNDZhNDJmLTNkNzMtMTFlOC1hYTM2LTAwMTU1ZDAxMGEzYSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZWZhdWx0OmRlZmF1bHQifQ.Qfi9881mVeX8Z4ophf6_l5e5jGcILFobS5mKTQgdoz6NF0rJ4buRASmQiqu4N5ErolOstSIJhhK1yVzHbkBYsYReip6ffTnOwF2cWU5EJAhP7_o2zGWK5b11amlp5qLU0rWucfYe34ZfGxAcxoekmgKwJ6Hu58JlgD3ae5lu-_J6yVT_O-klC6FUXCY-r3FbtwwYz7WbrSxhuu5nCbegmEy5gPy9aEeVZcz6v5ZIyTU62mvbO_M1xYQfPyaHPWgjbkh9H540j2LUa7Y7RQw_LLEp_NbpzVfN58sFLY8cndzUfr5v_KjtFXyPVqmUX0qxoIRWL2opB7Qr2lL7qyTg5

```







## 参考



- [Generate self-signed certificates](https://coreos.com/os/docs/latest/generate-self-signed-certificates.html)

- [Setting up a Certificate Authority and Creating TLS Certificates](https://github.com/kelseyhightower/kubernetes-the-hard-way/blob/master/docs/02-certificate-authority.md)

- [Client Certificates V/s Server Certificates](https://blogs.msdn.microsoft.com/kaushal/2012/02/17/client-certificates-vs-server-certificates/)

- [数字证书及 CA 的扫盲介绍](http://blog.jobbole.com/104919/)

- [kubernetes 1.9.1](https://jicki.me/kubernetes/2018/01/23/kubernetes-1.9.1.html)

- [kubernetes安装之证书认证](https://jimmysong.io/posts/kubernetes-tls-certificate/)

- [管理集群中的TLS](https://jimmysong.io/kubernetes-handbook/guide/managing-tls-in-a-cluster.html)

