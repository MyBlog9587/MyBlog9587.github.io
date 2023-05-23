---
title: kubernetes常用命令
date: 2023-05-23 16:06:04
categories: 
- kubernetes
- 运维
tags:
- kubernetes
---

# 创建版本k8s
`kubeadm init --kubernetes-version=v1.23.5 --pod-network-cidr=192.168.56.0/24`

# 通过调试日志查看 api 调用
`kubectl --v=8 get pods`

# 查看 api 支持版本以及资源对象
`kubectl api-versions` 或 `kubectl api-resources`

`kubectl api-resources --api-group=storage.k8s.io`

# 查看集群地址及凭证
`kubectl config view`

# 查看当前集群 主机列表
`kubectl get nodes`


# 通过 curl 访问 ApiServer
```bash
# APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
# TOKEN=$(kubectl get secret $(kubectl get serviceaccount default -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 --decode )
# curl $APISERVER/api --header "Authorization: Bearer $TOKEN" --insecure
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.2.15:6443"
    }
  ]
}
```

每次需要指定 TOKEN 很麻烦，可以通过使用 kubectl 代理的方式访问 ApiServer
```bash
[root@master ~]# kubectl proxy --port=8080 &
[1] 21533
[root@master ~]# Starting to serve on 127.0.0.1:8080

[root@master ~]# curl http://localhost:8080/api/
{
  "kind": "APIVersions",
  "versions": [
    "v1"
  ],
  "serverAddressByClientCIDRs": [
    {
      "clientCIDR": "0.0.0.0/0",
      "serverAddress": "10.0.2.15:6443"
    }
  ]
}
```

在 Kubernetes 中，要定位一个资源对象，需要指定 `GVR` 或 `GVK`，也就是`Group`、`Version`、`Reource`(即 `Kind`)

通过 `kubectl get pod -o wide -v 8` 可以看到请求 `GET https://10.0.2.15:6443/apis/apps/v1/namespaces/default/pod?limit=500`； 实际等同于使用 `curl http://localhost:8080/api/v1/namespaces/default/pods` API请求。


# 创建pod
`kubectl run kubernetes-bootcamp \ --image=docker.io/jocatalin/kubernetes-bootcamp:v1 \ --port=8080`

# 将容器内端口进行外部映射
`kubectl expose deployment/kubernetes-bootcamp \ --type="NodePort" \ --port 8080`

`kubectl expose deployment nginx --port=暴露端口 --target-port=映射端口 --type="NodePort\ClusterIP"`

# 查看应用被映射到节点的哪个端口
`kubectl get services`

# 查看副本数
`kubectl get deployments `

> 默认情况下应用只会运行一个副本

# 动态增加副本数
`kubectl scale deployments/kubernetes-bootcamp --replicas=3`

# 动态更新pod 镜像
`kubectl set image deployments/kubernetes-bootcamp kubernetes-bootcamp=jocatalin/kubernetes-bootcamp:v2`

# pod回退
`kubectl rollout undo deployments/kubernetes-bootcamp`
`kubectl rollout undo deployment nginx --to-revision=1`

> 默认回退到v1版本

# Controller 分类
包括 Deployment 、 ReplicaSet 、 DaemonSet 、 StatefuleSet 、 Job 等

# service作用
Service 为 Pod 提供了负载均衡

# Kubernetes 默认创建了两个 Namespace
- default ：创建资源时如果不指定，将被放到这个 Namespace 中。 
- kube-system ：Kubernetes 自己创建的系统资源将放到这个 Namespace 中

# kubectl命令自动补全配置
`echo "source <(kubectl completion bash)" >> ~/.bashrc`

# 查看初始化时的token(kubeadm init输出的)
`kubeadm token list`


# token 过期或者忘记时查询
```bash
# kubeadm token create --print-join-command
kubeadm join 10.0.4.15:6443 --token eor7pr.fjduq50rq0jlbhit --discovery-token-ca-cert-hash sha256:95bba3d6fa9690dc7035dde37847b90c9ea17402eafd0e7937cf276fc17814ce
```

# 查看pod状态
`kubectl describe pod`


# 查看所有pod，nodes中内存，CPU使用情况
## 查看pod
`kubectl top pod -n namespace_name`

## 查看具体某一个pod
`kubectl top pod pod_name -n ns_name`

## 查看node
`kubectl top nodes`

## 查看具体node
`kubectl top nodes node_name`

# 查看node 和 pod 对应关系
` kubectl get pod -o=custom-columns=NAME:.metadata.name,STATUS:.status.phase,NODE:.spec.nodeName,NS:.metadata.namespace --all-namespaces`

或

`kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=<node>`

# 将新节点加入集群
`kubeadm join --token xxxxx 192.168.56.105:6443`


# 获取所有节点
`kubectl get nodes`


# 给节点打标签
## 添加标签
`kubectl label node 节点名 node-role.kubernetes.io/标签名='标签值(可为空)'`

```bash
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   32m   v1.23.5
node1    Ready    <none>                 29m   v1.23.5
node2    Ready    <none>                 28m   v1.23.5
[root@master ~]#
[root@master ~]# kubectl label node node1 node-role.kubernetes.io/woker='node1'
node/node1 labeled
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   34m   v1.23.5
node1    Ready    woker                  31m   v1.23.5
node2    Ready    <none>                 30m   v1.23.5
```

## 删除标签
`kubectl label node 节点名 node-role.kubernetes.io/标签名-'`
```bash
[root@master ~]# kubectl label node node1 node-role.kubernetes.io/woker-
node/node1 unlabeled
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES                  AGE   VERSION
master   Ready    control-plane,master   39m   v1.23.5
node1    Ready    <none>                 36m   v1.23.5
node2    Ready    <none>                 35m   v1.23.5
```

## 修改标签
`kubectl label 类别(node\pod...) 资源名 --overwrite 标签名=新值`


# 设置ipvs模式
k8s集群为了访问互通，默认使用 `iptables`，大批量时会造成性能问题（kube-proxy 会在集群之间同步iptables的内容）

## 查看默认kube-proxy 使用模式
`kubectl logs -n kube-system kube-proxy-msb4c`
```yaml
I0519 08:20:44.949792       1 node.go:163] Successfully retrieved node IP: 192.168.56.130
I0519 08:20:44.949895       1 server_others.go:138] "Detected node IP" address="192.168.56.130"
I0519 08:20:44.956617       1 server_others.go:561] "Unknown proxy mode, assuming iptables proxy" proxyMode=""
I0519 08:20:45.199283       1 server_others.go:206] "Using iptables Proxier"
I0519 08:20:45.199362       1 server_others.go:213] "kube-proxy running in dual-stack mode" ipFamily=IPv4
I0519 08:20:45.199399       1 server_others.go:214] "Creating dualStackProxier for iptables"
I0519 08:20:45.199452       1 server_others.go:491] "Detect-local-mode set to ClusterCIDR, but no IPv6 cluster CIDR defined, , defaulting to no-op detect-local for IPv6"
I0519 08:20:45.200123       1 server.go:656] "Version info" version="v1.23.5"
I0519 08:20:45.202789       1 conntrack.go:52] "Setting nf_conntrack_max" nf_conntrack_max=131072
I0519 08:20:45.219264       1 config.go:317] "Starting service config controller"
I0519 08:20:45.219306       1 shared_informer.go:240] Waiting for caches to sync for service config
I0519 08:20:45.219360       1 config.go:226] "Starting endpoint slice config controller"
I0519 08:20:45.219374       1 shared_informer.go:240] Waiting for caches to sync for endpoint slice config
I0519 08:20:45.321348       1 shared_informer.go:247] Caches are synced for endpoint slice config
I0519 08:20:45.321448       1 shared_informer.go:247] Caches are synced for service config
```

## 修改 kube-proxy 配置文件 
修改 `mode: "ipvs"` 默认 iptables 集群打了以后很慢

`kubectl edit configmap kube-proxy -n kube-system`
```yaml
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
    mode: "ipvs"
    nodePortAddresses: null
```
修改完可以重启 kube-proxy 生效： 
```bash
# kubectl get pod -A -owide
# kubectl delete pod kube-proxy-5jzbn -n kube-system`
```

# 快速创建pod
```bash
# kubectl create deployment my-nginx --image=nginx
# kubectl get pod -A -o wide
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE     IP               NODE     NOMINATED NODE   READINESS GATES
default       my-nginx-c54945c55-wddw4         1/1     Running   0          2m28s   192.168.1.3      node1    <none>           <none>
```
## 访问测试
```bash
# curl 192.168.1.3 
# kubectl exec -ti my-nginx-c54945c55-wddw4 -- bash
root@my-nginx-c54945c55-wddw4:/# curl 192.168.1.3
```

# kubelet 特殊形式
kubelet 是唯一没有以容器形式运行的 Kubernetes 组件，它在 Ubuntu 中通过 Systemd 服务运行

# kubectl get 获取途径
执行 `kubectl get pod` 时 API Server 会从 etcd 中读取这些数据

# 将master变为node部署
出于安全考虑，默认配置下 Kubernetes 不会将 Pod 调度到 Master 节点。

- 如果希望将 k8s-master 也当作 Node 使用，可以执行如下命令： 

`kubectl taint node k8s-master node-role.kubernetes.io/master-`

- 如果要恢复 Master Only 状态，执行如下命令： 

`kubectl taint node k8s-master node-role.kubernetes.io/master="":NoSchedule`


# 默认dp配置 故障转移策略
Failover

# daemonset 特点
每个node 上最多只能运行一个副本

# 查看已结束pod
为 Pod 执行完毕后容器已经退出，需要用 `--show-all` 才能查看 Completed 状态的 Pod

# cron job 开启选项
Kubernetes 默认没有 enable CronJob 功能，需要在 kube-apiserver 中加入这个功能。方法很简单，修改 kube-apiserver 的配置文件 `/etc/kubernetes/manifests/kubeapiserver.yaml`


# cron job启动参数设置
`--runtime-config=batch/v2alpha1=true`


# 查看开启配置api server
`kubectl api-versions`

# svc的作用
Kubernetes Service 从逻辑上代表了一组 Pod ，具体是哪些 Pod 则是由 label 来挑选的。 Service 有自己的 IP ，而且这个 IP 是不变的。客户端只需要访问 Service 的 IP ， Kubernetes 则负责建立和维护 Service 与 Pod 的映射关系。

# svc的cluster ip 来源
Cluster IP 是一个虚拟 IP ，是由 Kubernetes 节点上的 iptables 规则管理的。可以通过 iptables-save 命令打印出当前节点的 iptables 规则

# 集群里svc一致
Cluster 的每一个节点都配置了相同的 iptables 规则，这样就确保了整个 Cluster 都能够通过 Service 的 Cluster IP 访问 Service

# 通过 kube-dns 访问
kube-dns 是一个 DNS 服务器。每当有新的 Service 被创建， kube-dns 会添加该 Service 的 DNS 记录。 Cluster 中的 Pod 可以通过 `<SERVICE_NAME>.<NAMESPACE_NAME>` 访问 Service 

## 完整域名格式
httpd-svc 的完整域: `httpd-svc.default.svc.cluster.local `

# 一个yml配置多个 注意点
多个资源可以在一个 YAML 文件中定义，用 `---` 分割

# 回滚参数配置
`--record` 的作用是将当前命令记录到 `revision` 记录中，这样我们就可以知道每个 `revison` 对应的是哪个配置文件了。通过 `kubectl rollout history deployment httpd` 查看 `revison` 历史记录

# 回滚命令
`kubectl rollout undo deployment httpd --to-revision=1`

# 回滚注意点
一定要在执行 `kubectl apply` 时加上 `--record` 参数

# 默认健康检查机制
每个容器启动时都会执行一个进程，此进程由 `Dockerfile` 的 `CMD` 或 `ENTRYPOINT` 指定。如果进程退出时返回码非零，则认为容器发生故障， Kubernetes 就会根据 `restartPolicy` 重启容器

# 健康检查另外两种方式liveness ，readiness区别
通过 `Liveness` 探测可以告诉 Kubernetes 什么时候通过重启容器实现自愈； `Readiness` 探测则是告诉 Kubernetes 什么时候可以将容器加入到 `Service` 负载均衡池中，对外提供服务

# liveness 和 readiness两种探活对比
1. Liveness 探测和Readiness 探测是两种Check 机制，如果不特意配置，Kubernetes 将对两种探测采取相同的默认行为，即通过判断容器启动进程的返回值是否为零来判断探测是否成功。
2. 两种探测的配置方法完全一样，支持的配置参数也一样。不同之处在于探测失败后的行为：Liveness 探测是重启容器；Readiness 探测则是将容器设置为不可用，不接收Service 转发的请求。
3. Liveness 探测和Readiness 探测是独立执行的，二者之间没有依赖，所以可以单独使用，也可以同时使用。用Liveness 探测判断容器是否需要重启以实现自愈；用Readiness 探测判断容器是否已经准备好对外提供服务。理解了Liveness 探测和Readiness 探测的原理，接下来讨论如何在业务场景中使用Check 。


# emptydir 属性以及有效期
一个 `emptyDir Volume` 是 `Host` 上的一个空目录。 `emptyDir Volume` 对于容器来说是持久的，对于 Pod 则不是。当 Pod 从节点删除时， Volume 的内容也会被删除。但如果只是容器被销毁而 Pod 还在，则 Volume 不受影响。也就是说： `emptyDir Volume` 的生命周期与 `Pod` 一致。


# emptydir使用场景
`emptyDir` 特别适合 Pod 中的容器需要临时共享存储空间的场景


# 查看容器配置命令
`docker inspect` 查看容器的详细配置信息

# hostpath应用原理及缺点
`hostPath Volume` 的作用是将 `Docker Host` 文件系统中已经存在的目录 mount 给 Pod 的容器。大部分应用都不会使用 `hostPath Volume`，因为这实际上增加了 Pod 与节点的耦合，限制了 Pod 的使用


# hostpath有效期
如果 `Pod` 被销毁了， `hostPath` 对应的目录还是会被保留


# pv的属性
`PersistentVolume`（ PV ）是外部存储系统中的一块存储空间，由管理员创建和维护。与 `Volume` 一样， PV 具有持久性，生命周期独立于 `Pod`

# pvc属性
`PersistentVolumeClaim`（ PVC ）是对 `PV` 的申请（ `Claim` ）。 `PVC` 通常由普通用户创建和维护。需要为 `Pod` 分配存储资源时，用户可以创建一个 `PVC` ，指明存储资源的容量大小和访问模式（比如只读）等信息， Kubernetes 会查找并提供满足条件的 `PV`


# pv读写权限三种模式
- `ReadWriteOnce`: 表示 `PV` 能以 `read-write` 模式 `mount` 到**单个**节点
- `ReadOnlyMany`：表示 `PV` 能以 `read-only` 模式 `mount` 到**多个**节点
- `ReadWriteMany`：表示 `PV` 能以 `read-write` 模式 `mount` 到**多个**节点

> 通过 `accessModes` 选项指定


# pv回收三种模式
- `Retain` 表示需要管理员手工回收； 
- `Recycle` 表示清除 `PV` 中的数据，效果相当于执行 `rm -rf/thevolume/*` ； 
- `Delete` 表示删除 `Storage Provider` 上的对应存储资源

# pv提供方式
- 静态供给
  提前创建了 `PV` ，然后通过 `PVC` 申请 `PV` 并在 `Pod` 中使用，这种方式叫作静态供给（ `Static Provision` ）

- 动态供给
  动态供给（ `Dynamical Provision` ），即如果没有满足 `PVC` 条件的 `PV` ，会动态创建 `PV` 。
  
相比静态供给，动态供给有明显的优势：不需要提前创建 `PV `，减少了管理员的工作量，效率高。动态供给是通过 `StorageClass` 实现的， `StorageClass` 定义了如何创建 PV


# pv动态供给 storageclass支持策略动作
`StorageClass` 支持 `Delete` 和 `Retain` 两种 `reclaimPolicy` ，默认是 `Delete`


# secret工作方式
`Secret` 会以密文的方式存储数据，避免了直接在配置文件中保存敏感信息。 `Secret` 会以 `Volume` 的形式被 `mount` 到 `Pod`


# secret 创建方式
- 通过 `--from-literal ：kubectl create secret generic mysecret --from-literal=username=admin --from-literal=password=123456` 每个--from-literal 对应一个信息条目。
- 通过 `--from-file ：echo -n admin > ./username echo -n 123456 > ./password kubectl create secret generic mysecret --from-file=./username --from-file=./password` 每个文件内容对应一个信息条目。
- 通过 `--from-env-file ：cat << EOF > env.txt username=admin password=123456 EOF kubectl create secret generic mysecret --from-env-file=env.txt` 文件env.txt 中每行Key=Value 对应一个信息条目。
- 通过 YAML 配置文件，文件中的敏感数据必须是通过 `base64` 编码后的结果


# 查看应用secret条数
通过 `kubectl describe secret` 查看条目的 Key

# 查看secret值
想查看 Value ，可以用 `kubectl edit secret mysecret`

# secret 使用方式
`Pod` 可以通过 `Volume` 或者环境变量的方式使用 `Secret`

## secret 文件方式优点
以 `Volume` 方式使用的 `Secret` 支持动态更新： `Secret` 更新后，容器中的数据也会更新

## secret 环境变量方式缺点
环境变量读取 `Secret` 很方便，但无法支撑 `Secret` 动态更新

# pod内容器共享资源以及pod间端口冲突解决问题
当 `Pod` 被调度到某个节点， `Pod` 中的所有容器都在这个节点上运行，这些容器共享相同的本地文件系统、 `IPC` 和网络命名空间。不同 `Pod` 之间不存在端口冲突的问题，因为每个 `Pod` 都有自己的 `IP` 地址。当某个容器使用 `localhost` 时，意味着使用的是容器所属 `Pod` 的地址空间


# pod 之间ip通信
`Pod` 的 `IP` 是集群可见的，即集群中的任何其他 `Pod` 和节点都可以通过 `IP` 直接与 `Pod` 通信，这种通信不需要借助任何网络地址转换、隧道或代理技术。 `Pod` 内部和外部使用的是同一个 `IP` ，这也意味着标准的命名服务和发现机制，比如 `DNS` 可以直接使用


# pod ip和service 作用范域
无论是 `Pod` 的 `IP` 还是 `Service` 的 `Cluster IP` ，它们只能在 `Kubernetes` 集群中可见，对集群之外的世界，这些 `IP` 都是私有的



# 获取 dashboard 密匙
`kubectl -n kubernetes-dashboard describe secrets $(kubectl -n kubernetes-dashboard get secrets | grep admin-user | awk '{print $1}')`


























