---
title: kubernetes源码调试
date: 2023-05-23 16:43:39
categories: [kubernetes, 源码]
tags:
- kubernetes
- private
---

# 源码调试

> [参考链接](https://mp.weixin.qq.com/s/TTcnXkrRgFUQ6W0KjWssXA)
>
> [远程debug 参考链接1](https://cloud.tencent.com/developer/article/1624638)
>
> [远程debug 参考链接2](https://duyanghao.github.io/kubernetes-remote-debug/)



# 本机调试：

## 1.查看当前pod状态:

```shell
[root@master ~]# kubectl get po -n kube-system
NAME                             READY   STATUS    RESTARTS       AGE
coredns-78fcd69978-6f4v9         1/1     Running   5 (43m ago)    4d2h
coredns-78fcd69978-w9lwn         1/1     Running   5 (43m ago)    4d2h
etcd-master                      1/1     Running   22 (43m ago)   4d2h
kube-apiserver-master            1/1     Running   22 (43m ago)   4d2h
kube-controller-manager-master   1/1     Running   7 (43m ago)    4d2h
kube-flannel-ds-rpd24            1/1     Running   5 (43m ago)    4d2h
kube-proxy-6d6tq                 1/1     Running   5 (43m ago)    4d2h
kube-scheduler-master            1/1     Running   7 (43m ago)    4d2h

```

## 2.挪开yaml文件，让scheduler停止：

```shell
[root@master ~]# mv /etc/kubernetes/manifests/kube-scheduler.yaml /home/
[root@master ~]# kubectl get po -n kube-system -o wide
NAME                             READY   STATUS    RESTARTS       AGE    IP                NODE     NOMINATED NODE   READINESS GATES
coredns-78fcd69978-6f4v9         1/1     Running   5 (46m ago)    4d3h   192.168.0.13      master   <none>           <none>
coredns-78fcd69978-w9lwn         1/1     Running   5 (46m ago)    4d3h   192.168.0.14      master   <none>           <none>
etcd-master                      1/1     Running   22 (46m ago)   4d3h   192.168.192.128   master   <none>           <none>
kube-apiserver-master            1/1     Running   22 (46m ago)   4d3h   192.168.192.128   master   <none>           <none>
kube-controller-manager-master   1/1     Running   7 (46m ago)    4d2h   192.168.192.128   master   <none>           <none>
kube-flannel-ds-rpd24            1/1     Running   5 (46m ago)    4d2h   192.168.192.128   master   <none>           <none>
kube-proxy-6d6tq                 1/1     Running   5 (46m ago)    4d3h   192.168.192.128   master   <none>           <none>
```

## 3.配置goland 调试代码

<img src="goland配置_1.png" alt="goland配置_1" style="zoom:80%;" />

### 3.1 构建配置
在main函数前面点一下这个绿色的三角形，当然这样运行肯定会失败，但是点一下会为我们生成一些配置，可以简化很多事情。点完之后开始配置：

<img src="debug_1.png" alt="debug" style="zoom:80%;" />

这里的Program arguments默认是空的，从前面挪动的 kube-scheduler.yaml 中可以看到如下配置：
```yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
...
```

### 3.2 设置断点
cmd/kube-scheduler/scheduler.go -> main()
```go
func main() {
	rand.Seed(time.Now().UnixNano())
	command := app.NewSchedulerCommand()
	...
```

### 3.3 debug

<img src="debug_2.png" alt="debug" style="zoom:80%;" />


### 3.4 交互

创建这个Deployment之后查看pod是pending状态

```shell
[root@master ~]# kubectl create -f k8s/tomcat-deploy.yaml 
deployment.apps/mytomcat created
[root@master ~]# 
[root@master ~]# 
[root@master ~]# kubectl get po
NAME                      READY   STATUS    RESTARTS   AGE
mytomcat-bb966477-7shgh   0/1     Pending   0          5s
[root@master ~]# 
```

### 3.5 调整断点(把断点打在scheduleOne()里面)

pkg/scheduler/scheduler.go -> scheduleOne()

```go
func (sched *Scheduler) scheduleOne(ctx context.Context) {
	podInfo := sched.NextPod()
	// pod could be nil when schedulerQueue is closed
	if podInfo == nil || podInfo.Pod == nil {
		return
	}
	pod := podInfo.Pod
	...
```

<img src="debug_3.png" alt="debug" style="zoom:100%;" />

### 3.6 查看pod 当前状态

```shell
[root@master ~]# kubectl get po
NAME                      READY   STATUS    RESTARTS   AGE
mytomcat-bb966477-7shgh   0/1     Pending   0          55m

```





# 远程调试：

**准备阶段**：

1. ***开发环境+golang环境+dlv+kubernetes源码*** 

   - 在远端linux服务器上安装一套kubernetes环境，可以用kubeadm工具安装； 

   - 安装golang环境+dlv+kubernetes源码，方便重编译k8s二进制
   - 默认k8s二进制开启了编译优化，忽略了debug信息；
   - dlv用来启个debug server提供给本地IDE远程调试使用
     - `go install github.com/go-delve/delve/cmd/dlv@latest`

2. ***修改编译参数***

   默认编译参数写死了加上-s -w选项 

   - **-s**: disable symbol table 禁用符号表 
   - **-w**: disable DWARF generation 禁用调试信息； 

   >  更多编译参数帮助信息查看：go tool link

   k8s.io/kubernetes/hack/lib/golang.sh` 中设置了`-s -w`选项来禁用符号表以及debug信息，因此在编译Kubernetes组件进行远程调试时需要去掉这两个限制。

   ```bash
   #goldflags="${GOLDFLAGS=-s -w -buildid=} $(kube::version::ldflags)"
   goldflags="${GOLDFLAGS=-buildid=} $(kube::version::ldflags)"
   ```

   

## 1. static pod组件

- kube-scheduler
- kube-controller
- kube-apiserver

**Static pod 组件debug 举例**

### 1.1 启动k8s:

```shell
[root@master kube-scheduler]# kubeadm init --pod-network-cidr=192.168.180.0/16
...

[root@master kube-scheduler]# kubectl get po -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-64897985d-dxvdq          1/1     Running   0          24m
coredns-64897985d-z229g          1/1     Running   0          24m
etcd-master                      1/1     Running   1          24m
kube-apiserver-master            1/1     Running   1          24m
kube-controller-manager-master   1/1     Running   8          24m
kube-flannel-ds-444pd            1/1     Running   0          22m
kube-proxy-g8tqm                 1/1     Running   0          24m
kube-scheduler-master            1/1     Running   1          24m
```



### 1.2 生成对应模块可执行文件：

```shell
[root@master ~]# cd $GOPATH/src/k8s.io/kubernetes/cmd/kube-scheduler/
[root@master kube-scheduler]# go build -v
[root@master kube-scheduler]# ls
app  BUILD  kube-scheduler  OWNERS  scheduler.go
```



### 1.3 查看对应模块参数配置：

> 默认k8s的static pod目录是: /etc/kubernetes/manifests/
>
> 看command部分就是启动命令+启动参数；

```shell
[root@master kube-scheduler]# cat /etc/kubernetes/manifests/kube-scheduler.yaml
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: null
  labels:
    component: kube-scheduler
    tier: control-plane
  name: kube-scheduler
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-scheduler
    - --authentication-kubeconfig=/etc/kubernetes/scheduler.conf
    - --authorization-kubeconfig=/etc/kubernetes/scheduler.conf
    - --bind-address=127.0.0.1
    - --kubeconfig=/etc/kubernetes/scheduler.conf
    - --leader-elect=true
    image: k8s.gcr.io/kube-scheduler:v1.23.1
```



### 1.4 移除对应模块：

> 找不到kube-scheduler container就说明已经停止

```shell
[root@master kube-scheduler]# mv /etc/kubernetes/manifests/kube-scheduler.yaml /root/k8s/
[root@master kube-scheduler]# kubectl get po -n kube-system
NAME                             READY   STATUS    RESTARTS   AGE
coredns-64897985d-dxvdq          1/1     Running   0          45m
coredns-64897985d-z229g          1/1     Running   0          45m
etcd-master                      1/1     Running   1          45m
kube-apiserver-master            1/1     Running   1          45m
kube-controller-manager-master   1/1     Running   8          45m
kube-flannel-ds-444pd            1/1     Running   0          43m
kube-proxy-g8tqm                 1/1     Running   0          45m

[root@master kube-scheduler]# docker ps -a |grep kube-scheduler
```



### 1.5 通过dlv方式启动模块：

```shell
[root@master kube-scheduler]# dlv exec ./kube-scheduler --listen=192.168.180.128:2345 --headless=true --api-version=2 --accept-multiclient -- \
  --authentication-kubeconfig=/etc/kubernetes/scheduler.conf \
  --authorization-kubeconfig=/etc/kubernetes/scheduler.conf \
  --bind-address=127.0.0.1 \
  --kubeconfig=/etc/kubernetes/scheduler.conf \
  --leader-elect=true
  
API server listening at: 192.168.180.128:2345
2022-01-13T15:19:27+08:00 warning layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)

```



### 1.6 打开Golang IDE， 添加Run/Debug Configurations，新添加一个Go Remote配置:

![远程debug](远程debug.png)

### 1.7 给kube-scheduler的启动函数设置断点，然后点击Debug(绿色小瓢虫那个按钮):

![kube-scheduler](kube-scheduler-debug.png)



## 2. daemonset+configmap配置文件方式

**kube-proxy的启动方式不是static pod方式；是以daemonset+configmap配置文件方式启动服务的；**

- **kube-proxy**



### 2.1 查看定义：

> 启动命令参数看daemonset中的command部分，配置文件看configmap中的config.conf和kubeconfig.conf

```shell
[root@master kube-scheduler]# kubectl -n kube-system get ds kube-proxy -o yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  annotations:
    deprecated.daemonset.template.generation: "1"
  creationTimestamp: "2022-01-13T06:44:00Z"
  generation: 1
  labels:
    k8s-app: kube-proxy
  name: kube-proxy
  namespace: kube-system
  resourceVersion: "459"
  uid: 1e3bf1f3-c9b3-4059-b34e-8a58842e5495
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: kube-proxy
  template:
    metadata:
      creationTimestamp: null
      labels:
        k8s-app: kube-proxy
    spec:
      containers:
      - command:
        - /usr/local/bin/kube-proxy
        - --config=/var/lib/kube-proxy/config.conf
        - --hostname-override=$(NODE_NAME)
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: spec.nodeName
        image: k8s.gcr.io/kube-proxy:v1.23.1
        imagePullPolicy: IfNotPresent
        name: kube-proxy
        resources: {}
        securityContext:
          privileged: true
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/lib/kube-proxy
          name: kube-proxy
        - mountPath: /run/xtables.lock
          name: xtables-lock
        - mountPath: /lib/modules
          name: lib-modules
          readOnly: true
      dnsPolicy: ClusterFirst
      hostNetwork: true
      nodeSelector:
        kubernetes.io/os: linux
      priorityClassName: system-node-critical
      ...
```



### 2.2 停止kube-proxy pod运行:

> 通过修改 nodeSelector 标签，让其匹配不上

```shell
[root@master kube-scheduler]# kubectl -n kube-system edit daemonsets. kube-proxy
nodeSelector:
  beta.kubernetes.io/os: linux2
```

为了不影响其它母机上的kube-proxy，可以通过设置node标签以及nodeSelector的方式将某一个节点的kube-proxy停止：

```shell
[root@master kube-scheduler]# kubectl label nodes <node-name> <label-key>=<label-value>
```



### 2.3 Proxy config.conf:

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
bindAddressHardFail: false
clientConnection:
  acceptContentTypes: ""
  burst: 0
  contentType: ""
  kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
  qps: 0
clusterCIDR: 192.168.180.0/16
configSyncPeriod: 0s
conntrack:
  maxPerCore: null
  min: null
  tcpCloseWaitTimeout: null
  tcpEstablishedTimeout: null
detectLocalMode: ""
enableProfiling: false
healthzBindAddress: ""
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: null
  minSyncPeriod: 0s
  syncPeriod: 0s
ipvs:
  excludeCIDRs: null
  minSyncPeriod: 0s
  scheduler: ""
  strictARP: false
  syncPeriod: 0s
  tcpFinTimeout: 0s
  tcpTimeout: 0s
  udpTimeout: 0s
kind: KubeProxyConfiguration
metricsBindAddress: ""
mode: ""
nodePortAddresses: null
oomScoreAdj: null
portRange: ""
showHiddenMetricsForVersion: ""
udpIdleTimeout: 0s
winkernel:
  enableDSR: false
  networkName: ""
  sourceVip: ""
```



### 2.4 dlv启动kube-proxy:

> **注意master 是 nodeName，替换成实际节点名称**

```shell
[root@master kube-proxy]# dlv exec ./kube-proxy --headless -l 192.168.180.128:2345 --api-version=2 --accept-multiclient -- \
  --config= /root/k8s/proxy/config.conf \
  --hostname-override=master
  
API server listening at: 192.168.180.128:2345
2022-01-13T18:21:54+08:00 warning layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)
```



### 2.5 debug kube-proxy：

![kube-proxy](kube-proxy-debug.png)



## 3. systemd管理

- ### kubelet



### 3.1 查看配置：

```shell
[root@master kubelet]# cat /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS
[root@master kubelet]#
```



### 3.2 查看启动参数：

```shell
[root@master kubelet]# ps aux|grep kubelet
root      10769  5.2  3.4 1469596 70964 ?       Ssl  21:48   0:09 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.6
root      11899  0.0  0.0 112728   976 pts/0    S+   21:51   0:00 grep --color=auto kubelet
root      18184  7.5 13.1 1111476 267288 ?      Ssl  14:43  32:23 kube-apiserver --advertise-address=192.168.248.131 --allow-privileged=true --authorization-mode=Node,RBAC --client-ca-file=/etc/kubernetes/pki/ca.crt --enable-admission-plugins=NodeRestriction --enable-bootstrap-token-auth=true --etcd-cafile=/etc/kubernetes/pki/etcd/ca.crt --etcd-certfile=/etc/kubernetes/pki/apiserver-etcd-client.crt --etcd-keyfile=/etc/kubernetes/pki/apiserver-etcd-client.key --etcd-servers=https://127.0.0.1:2379 --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname --proxy-client-cert-file=/etc/kubernetes/pki/front-proxy-client.crt --proxy-client-key-file=/etc/kubernetes/pki/front-proxy-client.key --requestheader-allowed-names=front-proxy-client --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt --requestheader-extra-headers-prefix=X-Remote-Extra- --requestheader-group-headers=X-Remote-Group --requestheader-username-headers=X-Remote-User --secure-port=6443 --service-account-issuer=https://kubernetes.default.svc.cluster.local --service-account-key-file=/etc/kubernetes/pki/sa.pub --service-account-signing-key-file=/etc/kubernetes/pki/sa.key --service-cluster-ip-range=10.96.0.0/12 --tls-cert-file=/etc/kubernetes/pki/apiserver.crt --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
[root@master kubelet]#
```



### 3.3 停止kubelet:

```shell
[root@master kubelet]# systemctl stop kubelet.service
```



### 3.4 dlv执行：

```shell
[root@master kubelet]# dlv exec ./kubelet --headless -l 192.168.180.128:2345 --api-version=2 --accept-multiclient -- \
  --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf \
  --kubeconfig=/etc/kubernetes/kubelet.conf \
  --config=/var/lib/kubelet/config.yaml \
  --cgroup-driver=cgroupfs \
  --hostname-override=192.168.180.128 \
  --network-plugin=cni \
  --pod-infra-container-image=k8s.gcr.io/pause:3.6

2022-01-13T21:59:23+08:00 warning layer=rpc Listening for remote connections (connections are not authenticated nor encrypted)
```

> 其中--cgroup-driver后面部分是/var/lib/kubelet/kubeadm-flags.env文件内容，而 KUBELET_EXTRA_ARGS 参数在/etc/sysconfig/kubelet文件中



### 3.5 Kubelet debug:

![kubelet-debug](kubelet-debug.png)





>  [dlv 参考](http://lday.me/2017/02/27/0005_gdb-vs-dlv/)


# windows 虚拟机共享文件夹方式 联调配置：
1. 使用管理员开启cmd
2. 创建软连接

```bash
cd k8s.io
cd kubernetes
mklink BUILD.bazel build\root\BUILD.root
mklink CHANGELOG.md  CHANGELOG\README.md
mklink Makefile build\root\Makefile
mklink Makefile.generated_files  build\root\Makefile.generated_files
mklink WORKSPACE build\root\WORKSPACE
cd vendor
cd k8s.io
mklink api  ..\..\staging\src\k8s.io\api
mklink apiextensions-apiserver  ..\..\staging\src\k8s.io\apiextensions-apiserver
mklink apimachinery ..\..\staging\src\k8s.io\apimachinery
mklink apiserver ..\..\staging\src\k8s.io\apiserver
mklink client-go ..\..\staging\src\k8s.io\client-go
mklink cli-runtime ..\..\staging\src\k8s.io\cli-runtime
mklink cloud-provider  ..\..\staging\src\k8s.io\cloud-provider
mklink cluster-bootstrap  ..\..\staging\src\k8s.io\cluster-bootstrap
mklink code-generator ..\..\staging\src\k8s.io\code-generator
mklink component-base  ..\..\staging\src\k8s.io\component-base
mklink controller-manager  ..\..\staging\src\k8s.io\controller-manager
mklink cri-api  ..\..\staging\src\k8s.io\cri-api
mklink csi-translation-lib  ..\..\staging\src\k8s.io\csi-translation-lib
mklink kube-aggregator ..\..\staging\src\k8s.io\kube-aggregator
mklink kube-controller-manager  ..\..\staging\src\k8s.io\kube-controller-manager
mklink kubectl ..\..\staging\src\k8s.io\kubectl
mklink kubelet ..\..\staging\src\k8s.io\kubelet
mklink kube-proxy ..\..\staging\src\k8s.io\kube-proxy
mklink kube-scheduler  ..\..\staging\src\k8s.io\kube-scheduler
mklink legacy-cloud-providers ..\..\staging\src\k8s.io\legacy-cloud-providers
mklink metrics  ..\..\staging\src\k8s.io\metrics
mklink sample-apiserver  ..\..\staging\src\k8s.io\sample-apiserver
mklink sample-cli-plugin ..\..\staging\src\k8s.io\sample-cli-plugin
mklink sample-controller  ..\..\staging\src\k8s.io\sample-controller
```