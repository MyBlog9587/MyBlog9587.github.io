---
title: kubernetes部署实践
date: 2023-05-23 16:09:29
categories:
- kubernetes
- 运维
tags:
- kubernetes
- private
---

# k8s 

## 部署参考:

---

### 单节点部署

[参考连接](https://mp.weixin.qq.com/s/NdxulImlipD-zWUY2eUHYw)

#### 网卡环境配置：

<img src="机器环境1.png" alt="机器环境1" style="zoom:50%;" />

<img src="机器环境2.png" alt="机器环境2" style="zoom:50%;" />

<img src="机器环境3.png" alt="机器环境3" style="zoom:50%;" />

<img src="机器环境4.png" alt="机器环境4" style="zoom:50%;" />

<img src="机器环境5.png" alt="机器环境5" style="zoom:50%;" />

#### 初始化:

```shell
[root@master ~]# kubeadm init --pod-network-cidr=192.168.101.0/16
I1213 17:05:28.509486   96937 version.go:255] remote version is much newer: v1.23.0; falling back to: stable-1.22
[init] Using Kubernetes version: v1.22.4
[preflight] Running pre-flight checks
...

[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.101.135:6443 --token p8gwg9.n4h8kfjymqjq2stg \
	--discovery-token-ca-cert-hash sha256:e4bcf02565b48d371fc7adcfabac3754667bb0909b6cd3213fb10d9ca05ce395 
```

##### kubeadm init 失败:

```shell
...
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.
[kubelet-check] It seems like the kubelet isn't running or healthy.
[kubelet-check] The HTTP call equal to 'curl -sSL http://localhost:10248/healthz' failed with error: Get "http://localhost:10248/healthz": dial tcp [::1]:10248: connect: connection refused.

```

##### 解决办法:

```shell
[root@master ~]# sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://dx9629ms.mirror.aliyuncs.com"]
}
EOF
[root@master ~]# systemctl daemon-reload
[root@master ~]# systemctl restart docker
[root@master ~]# kubeadm reset
[root@master ~]# kubeadm init ...
```


```shell
[root@master ~]# mkdir -p $HOME/.kube
[root@master ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@master ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config
[root@master ~]# export KUBECONFIG=/etc/kubernetes/admin.conf

[root@master ~]# kubectl get node
NAME     STATUS     ROLES                  AGE   VERSION
master   NotReady   control-plane,master   15m   v1.22.4
[root@master ~]# 

```

#### 构建网络:

```shell
[root@master ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
或者
[root@master ~]# kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/k8s-manifests/kube-flannel-rbac.yml
[root@master ~]# kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
serviceaccount/flannel created
...
````

##### flannel (如果上面自定义了pod ip范围，这里需要修改flannel的configmap), flannel的Network修改：默认是10.244.0.0/16, pod的cidr是192.168.101.0/16，所以下面也改成192.168.101.0/16

```shell
[root@master ~]# cat /run/flannel/subnet.env 
FLANNEL_NETWORK=10.244.0.0/16
FLANNEL_SUBNET=192.168.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true

[root@master ~]# kubectl delete  -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
[root@master ~]# wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
[root@master ~]# vim kube-flannel.yml
...
  net-conf.json: |
    {
      "Network": "192.168.101.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
...

[root@master ~]# kubectl create -f kube-flannel.yml
[root@master ~]# cat /run/flannel/subnet.env 
FLANNEL_NETWORK=192.168.0.0/16
FLANNEL_SUBNET=192.168.0.1/24
FLANNEL_MTU=1450
FLANNEL_IPMASQ=true
[root@master ~]# 
[root@master ~]# kubectl get cm kube-flannel-cfg -n kube-system -o yaml
apiVersion: v1
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "192.168.101.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"cni-conf.json":"{\n  \"name\": \"cbr0\",\n  \"cniVersion\": \"0.3.1\",\n  \"plugins\": [\n    {\n      \"type\": \"flannel\",\n      \"delegate\": {\n        \"hairpinMode\": true,\n        \"isDefaultGateway\": true\n      }\n    },\n    {\n      \"type\": \"portmap\",\n      \"capabilities\": {\n        \"portMappings\": true\n      }\n    }\n  ]\n}\n","net-conf.json":"{\n  \"Network\": \"192.168.101.0/16\",\n  \"Backend\": {\n    \"Type\": \"vxlan\"\n  }\n}\n"},"kind":"ConfigMap","metadata":{"annotations":{},"labels":{"app":"flannel","tier":"node"},"name":"kube-flannel-cfg","namespace":"kube-system"}}
  creationTimestamp: "2021-12-14T07:58:24Z"
  labels:
    app: flannel
    tier: node
  name: kube-flannel-cfg
  namespace: kube-system
  resourceVersion: "534"
  uid: 234aff90-e261-47d3-81d1-4f4563bc41bf

[root@master ~]# 

```

#### flannel 构建完 等待 node read:

```shell
[root@master ~]# kubectl get node
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   10m   v1.22.4
[root@master ~]# kubectl get pod -n kube-system
NAME                             READY   STATUS              RESTARTS   AGE
coredns-78fcd69978-ld6jp         0/1     ContainerCreating   0          11m
coredns-78fcd69978-mvl9v         0/1     ContainerCreating   0          11m
etcd-master                      1/1     Running             0          11m
kube-apiserver-master            1/1     Running             0          11m
kube-controller-manager-master   1/1     Running             0          11m
kube-proxy-w99k6                 1/1     Running             0          11m
kube-scheduler-master            1/1     Running             0          11m
[root@master ~]# 
```

##### coredns ContainerCreating 原因：

```shell
[root@master ~]# kubectl describe po coredns-78fcd69978-dz646 -n kube-system
Name:                 coredns-78fcd69978-dz646
Namespace:            kube-system
Priority:             2000000000
Priority Class Name:  system-cluster-critical
Node:                 master/192.168.192.128
Start Time:           Tue, 14 Dec 2021 15:57:32 +0800
Labels:               k8s-app=kube-dns
                      pod-template-hash=78fcd69978
Annotations:          <none>
Status:               Pending
IP:                   
IPs:                  <none>
Controlled By:        ReplicaSet/coredns-78fcd69978
Containers:
  coredns:
    Container ID:  
    Image:         k8s.gcr.io/coredns/coredns:v1.8.4
    Image ID:      
    Ports:         53/UDP, 53/TCP, 9153/TCP
    Host Ports:    0/UDP, 0/TCP, 0/TCP
    Args:
      -conf
      /etc/coredns/Corefile
    State:          Waiting
      Reason:       ContainerCreating
    Ready:          False
    Restart Count:  0
    Limits:
      memory:  170Mi
    Requests:
      cpu:        100m
      memory:     70Mi
    Liveness:     http-get http://:8080/health delay=60s timeout=5s period=10s #success=1 #failure=5
    Readiness:    http-get http://:8181/ready delay=0s timeout=1s period=10s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /etc/coredns from config-volume (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-kwdb4 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  config-volume:
    Type:      ConfigMap (a volume populated by a ConfigMap)
    Name:      coredns
    Optional:  false
  kube-api-access-kwdb4:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Burstable
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 CriticalAddonsOnly op=Exists
                             node-role.kubernetes.io/control-plane:NoSchedule
                             node-role.kubernetes.io/master:NoSchedule
                             node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason                  Age                   From               Message
  ----     ------                  ----                  ----               -------
  Normal   Scheduled               46m                   default-scheduler  Successfully assigned kube-system/coredns-78fcd69978-dz646 to master
  Warning  FailedCreatePodSandBox  46m                   kubelet            Failed to create pod sandbox: rpc error: code = Unknown desc = [failed to set up sandbox container "e816d1d5585f72b2a48751fd39129c821876439e14d98b1ed317def887a1fc06" network for pod "coredns-78fcd69978-dz646": networkPlugin cni failed to set up pod "coredns-78fcd69978-dz646_kube-system" network: error getting ClusterInformation: Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes"), failed to clean up sandbox container "e816d1d5585f72b2a48751fd39129c821876439e14d98b1ed317def887a1fc06" network for pod "coredns-78fcd69978-dz646": networkPlugin cni failed to teardown pod "coredns-78fcd69978-dz646_kube-system" network: error getting ClusterInformation: Get "https://10.96.0.1:443/apis/crd.projectcalico.org/v1/clusterinformations/default": x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")]
  Normal   SandboxChanged          41m (x25 over 46m)    kubelet            Pod sandbox changed, it will be killed and re-created.
  Normal   SandboxChanged          102s (x161 over 36m)  kubelet            Pod sandbox changed, it will be killed and re-created.
[root@master ~]# 

```

##### 解决办法：

```shell
[root@master ~]# kubeadm reset
[root@master ~]# rm -rf /etc/cni/net.d
[root@master ~]# kubeadm init --pod-network-cidr=192.168.101.0/16

[root@master ~]# mkdir -p $HOME/.kube
[root@master ~]# sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@master ~]# sudo chown $(id -u):$(id -g) $HOME/.kube/config

设置一次就够：
[root@master ~]# echo export KUBECONFIG=/etc/kubernetes/admin.conf >> ~/.bashrc
[root@master ~]# echo export KUBECONFIG=/etc/kubernetes/admin.conf >> /etc/profile
[root@master ~]# source ~/.bashrc
[root@master ~]# source /etc/profile
[root@master ~]# echo $KUBECONFIG
/etc/kubernetes/admin.conf

[root@master ~]# kubectl get node
NAME     STATUS     ROLES                  AGE   VERSION
master   NotReady   control-plane,master   15m   v1.22.4
[root@master ~]# kubectl create -f kube-flannel.yml

[root@master ~]# kubectl get node
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   16m   v1.22.4
[root@master ~]# kubectl get pod -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-78fcd69978-bz8c9         1/1     Running   0          16m
coredns-78fcd69978-x6vmq         1/1     Running   0          16m
etcd-master                      1/1     Running   11         16m
kube-apiserver-master            1/1     Running   10         16m
kube-controller-manager-master   1/1     Running   1          16m
kube-flannel-ds-v4jsz            1/1     Running   0          20s
kube-proxy-t8s9s                 1/1     Running   0          16m
kube-scheduler-master            1/1     Running   1          16m

```

#### 接之前的操作：

```shell
[root@master ~]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
etcd-0               Healthy     {"health":"true","reason":""}                                                                 
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
controller-manager   Healthy     ok                                                                                            
```

##### 异常处理: 出现这种情况，是/etc/kubernetes/manifests/下的kube-controller-manager.yaml和kube-scheduler.yaml设置的默认端口是0 (--port=0) 导致的，解决方式是注释掉对应的port即可

```shell
[root@master ~]# vim /etc/kubernetes/manifests/kube-scheduler.yaml 
[root@master ~]# vim /etc/kubernetes/manifests/kube-controller-manager.yaml 
[root@master ~]# systemctl restart kubelet.service
[root@master ~]# 
[root@master ~]# kubectl get cs
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE                         ERROR
scheduler            Healthy   ok                              
controller-manager   Healthy   ok                              
etcd-0               Healthy   {"health":"true","reason":""}   
[root@master ~]# 

```

```shell
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   12m   v1.22.4
[root@master ~]# kubectl describe node master | grep Taints
Taints:             node-role.kubernetes.io/master:NoSchedule
```

##### 默认master节点是不跑业务pod的，我们暂时只有一个node，所以先去掉这个Taint：

```shell
[root@master ~]# kubectl taint nodes master node-role.kubernetes.io/master-
node/master untainted
[root@master ~]# kubectl describe node master | grep Taints
Taints:             <none>
[root@master ~]# 

```

##### master 上添加taint:

`kubectl taint nodes master1 node-role.kubernetes.io/master=:NoSchedule`

##### 取消master1 上的 taint:

`kubectl taint nodes master1 node-role.kubernetes.io/master-`


#### 构建测试pod: tomcat-deploy.yaml 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mytomcat
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mytomcat
  template:
    metadata:
      name: mytomcat
      labels:
        app: mytomcat
    spec:
      containers:
      - name: mytomcat
        image: tomcat:8
        ports:
        - containerPort: 8080
```

##### 观察进度:

```shell
[root@master ~]# kubectl create -f tomcat-deploy.yaml 
deployment.apps/mytomcat created
[root@master ~]# 
[root@master ~]# kubectl get po
NAME                      READY   STATUS              RESTARTS   AGE
mytomcat-bb966477-cp5zr   0/1     ContainerCreating   0          7s

ot@master ~]# kubectl describe po mytomcat-bb966477-cp5zr
Name:         mytomcat-bb966477-cp5zr
Namespace:    default
Priority:     0
Node:         master/192.168.192.128
Start Time:   Tue, 14 Dec 2021 17:42:20 +0800
Labels:       app=mytomcat
              pod-template-hash=bb966477
Annotations:  <none>
Status:       Running
IP:           192.168.0.4
IPs:
  IP:           192.168.0.4
Controlled By:  ReplicaSet/mytomcat-bb966477
Containers:
  mytomcat:
    Container ID:   docker://df0944d8f77ed8ce56a440b89f6ed1ce0e3ea5e3dc13652ee5d330e5c857b4f6
    Image:          tomcat:8
    Image ID:       docker-pullable://tomcat@sha256:bffc0465384fa41e5dff9aceb8435dc65e6b3eb6ad1b9b30bacf72e1d03dfe9d
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Tue, 14 Dec 2021 17:43:09 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-v8qc9 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-v8qc9:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  52s   default-scheduler  Successfully assigned default/mytomcat-bb966477-cp5zr to master
  Normal  Pulling    52s   kubelet            Pulling image "tomcat:8"
  Normal  Pulled     5s    kubelet            Successfully pulled image "tomcat:8" in 47.56973485s
  Normal  Created    4s    kubelet            Created container mytomcat
  Normal  Started    4s    kubelet            Started container mytomcat
[root@master ~]# 
[root@master ~]# 
[root@master ~]# kubectl get po
NAME                      READY   STATUS    RESTARTS   AGE
mytomcat-bb966477-cp5zr   1/1     Running   0          59s

```

##### 验证是否可用:

```shell
[root@master ~]# kubectl get pod -o wide
NAME                      READY   STATUS    RESTARTS   AGE     IP            NODE     NOMINATED NODE   READINESS GATES
mytomcat-bb966477-cp5zr   1/1     Running   0          3m47s   192.168.0.4   master   <none>           <none>
[root@master ~]# 
[root@master ~]# curl 192.168.0.4:8080
````


---


### 集群部署

[参考链接](https://mp.weixin.qq.com/s/Bxsi5UUxfy2G8qN6yEJdjA)


---

## 快捷配置：
/etc/profile
```bash
export KUBECONFIG=/etc/kubernetes/admin.conf

export GOROOT=/usr/local/go
export GOPATH=/root/go_work
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
```

/etc/bashrc 
```bash
alias cp='cp -i'
alias egrep='egrep --color=auto'
alias fgrep='fgrep --color=auto'
alias grep='grep --color=auto'
alias kc='kubectl create -f'
alias kd='kubectl describe'
alias kg='kubectl get'
alias ks='kubectl -n kube-system'
alias kt='kubeadm token'
alias l.='ls -d .* --color=auto'
alias ll='ls -l --color=auto'
alias ls='ls --color=auto'
alias mv='mv -i'
alias rm='rm -i'
alias which='alias | /usr/bin/which --tty-only --read-alias --show-dot --show-tilde'
```

---

## 常用命令
### get token : `kubeadm token list`
### get ca-cert-hash: `openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null |openssl dgst -sha256 -hex | sed 's/^.* //'`

### init

`kubeadm join 192.168.192.128:6443 --token 6eoyps.9tmgii4f9jn0lwyt --discovery-token-ca-cert-hash sha256:5021e9aa11cec63dca94e156b0eff371cf76462dca81df9a6279ee1cc375f4d6`

`kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.22.0 --apiserver-advertise-address=192.168.192.128`

`kubeadm init --image-repository registry.aliyuncs.com/google_containers --kubernetes-version v1.22.0 --apiserver-advertise-address=192.168.192.128 --pod-network-cidr=192.168.101.0/16`

初始化失败:

```shell
[root@master ~]# kubeadm init --pod-network-cidr=192.168.101.0/16
I1213 16:50:27.921394   96094 version.go:255] remote version is much newer: v1.23.0; falling back to: stable-1.22
[init] Using Kubernetes version: v1.22.4
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher

```
解决办法:
`echo 1 > /proc/sys/net/ipv4/ip_forward`



### watch

`watch kubectl get pods -n calico-system`

`watch kubectl get nodes`


### cluster info

`kubectl cluster-info`

```shell
[root@vm1 ~]# kubectl cluster-info
Kubernetes control plane is running at https://192.168.192.128:6443
CoreDNS is running at https://192.168.192.128:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

### 异常处理 

#### https://192.168.192.128:6443/ web error

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403
}
```

生成client-certificate-data

`grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt`
生成client-key-data

`grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key`
生成p12

`openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"`
kubecfg.p12就是生成的个人证书

浏览器导入证书,然后关闭浏览器，重新登录后通过token登录.
```shell
[root@master ~]# kubectl get secret -n kube-system |grep admin|awk '{print $1}'
dashboard-admin-token-swhrz
[root@master ~]# kubectl describe secret/dashboard-admin-token-swhrz -n kube-system
Name:         dashboard-admin-token-swhrz
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: dashboard-admin
              kubernetes.io/service-account.uid: d59adb94-f4ae-4180-8b69-4cd8f2c2e5f4

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Ikp2bV9pZmNIR0xqLUxRREd3QlRzNU1pdnBkYnMxTXRlWG15alBidW0xNTAifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkYXNoYm9hcmQtYWRtaW4tdG9rZW4tc3docnoiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC5uYW1lIjoiZGFzaGJvYXJkLWFkbWluIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiZDU5YWRiOTQtZjRhZS00MTgwLThiNjktNGNkOGYyYzJlNWY0Iiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50Omt1YmUtc3lzdGVtOmRhc2hib2FyZC1hZG1pbiJ9.K0td6E4SjkgjQvQ9ucxecNkhEFmKhOtrwlgNpq2yJZvdm_MOuSAl4P7J7PGkFf6UoEXJ1jgk4eyMeLR9eJZ8KV9rwTt5U-snH_dGetejeofI6pk0aIHWyIq7KnuKbH8m_Q8Ok4eDatOW06_Q8hs0ZYktZ-J5uPytuS0jUuG47pxRTu5PwFtR-svypE7mP7Sz1rORyT7wultWysvA1zFS93DhRlIBJwbvv2UQI9cDbJcXl3x-HItPpZaPFrGqKTRZoXvAxoaUCm7BhPm9XO0xhE5H_ItGO09IZnb_Ib3kCF-W9-9fITPBIo4vaF9Z7m7nbaz9StID2RrCWV7iP1ysgg
```






