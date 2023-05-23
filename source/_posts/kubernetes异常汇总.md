---
title: kubernetes异常汇总
date: 2023-05-23 16:08:34
categories:
- kubernetes
- 运维
tags:
- kubernetes
---

1. 执行kubctl 命令提示：`The connection to the server 10.0.2.15:6443 was refused - did you specify the right host or port?`

排查方法：
`journalctl -fu kubelet`

查看异常日志
```bash
Mar 24 10:04:00 master kubelet[25271]: I0324 10:04:00.410575   25271 server.go:446] "Kubelet version" kubeletVersion="v1.23.5"
Mar 24 10:04:00 master kubelet[25271]: I0324 10:04:00.411264   25271 server.go:874] "Client rotation is on, will bootstrap in background"
Mar 24 10:04:00 master kubelet[25271]: I0324 10:04:00.415985   25271 certificate_store.go:130] Loading cert/key pair from "/var/lib/kubelet/pki/kubelet-client-current.pem".
Mar 24 10:04:00 master kubelet[25271]: I0324 10:04:00.418225   25271 dynamic_cafile_content.go:156] "Starting controller" name="client-ca-bundle::/etc/kubernetes/pki/ca.crt"
Mar 24 10:04:00 master kubelet[25271]: I0324 10:04:00.513272   25271 server.go:693] "--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /"
Mar 24 10:04:00 master kubelet[25271]: E0324 10:04:00.513421   25271 server.go:302] "Failed to run kubelet" err="failed to run Kubelet: running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/swaps contained: [Filename\t\t\t\tType\t\tSize\tUsed\tPriority /dev/dm-1                               partition\t2097148\t0\t-2]"
Mar 24 10:04:00 master systemd[1]: kubelet.service: main process exited, code=exited, status=1/FAILURE
Mar 24 10:04:00 master systemd[1]: Unit kubelet.service entered failed state.
Mar 24 10:04:00 master systemd[1]: kubelet.service failed.
Mar 24 10:04:10 master systemd[1]: kubelet.service holdoff time over, scheduling restart.
Mar 24 10:04:10 master systemd[1]: Stopped kubelet: The Kubernetes Node Agent.
Mar 24 10:04:10 master systemd[1]: Started kubelet: The Kubernetes Node Agent.
```

排查方法：
通过日志查看提示 异常是 `err="failed to run Kubelet: running with swap on is not supported, please disable swap`

通过确认，确实是swap没有关闭成功导致

```bash
[root@master ~]# free -h
              total        used        free      shared  buff/cache   available
Mem:           2.9G        946M        100M         83M        1.9G        1.7G
Swap:          2.0G          0B        2.0G
[root@master ~]# 
```
解决办法1：

关闭swap
```bash
[root@master ~]# swapoff  -a
[root@master ~]# sed -ri 's/.*swap.*/#&/' /etc/fstab
```

解决办法2:

通过参数忽略swap报错, 在kubeadm初始化时增加 `--ignore-preflight-errors=Swap` 参数，注意Swap中S要大写

```bash
# kubeadm init --ignore-preflight-errors=Swap
```

另外还要设置/etc/sysconfig/kubelet参数
```bash
# sed -i 's/KUBELET_EXTRA_ARGS=/KUBELET_EXTRA_ARGS="--fail-swap-on=false"/' /etc/sysconfig/kubelet
```

暴力解决办法：

```bash
sudo -i
swapoff -a
exit
strace -eopenat kubectl version
```


2. kubeadm 初始化告警 `[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/`

两种解决方式：

一、编辑docker配置文件`/etc/docker/daemon.json`
```bash
"exec-opts": ["native.cgroupdriver=systemd"]
```

```bash
systemctl daemon-reload
systemctl restart docker
```
二、编辑`/usr/lib/systemd/system/docker.service`
```bash
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --exec-opt native.cgroupdriver=systemd
```

```bash
systemctl daemon-reload
systemctl restart docker
```

设置完成后通过docker info命令可以看到Cgroup Driver为systemd
```bash
docker info | grep Cgroup
```

3. kubeadm join 报证书过期问题 `... certificate has expired or is not yet valid To see the stack trace of this error execute with --v=5 or higher`
1、时间错误

本地时间错误同样会导致证书过期报错，请检查本地时间是否正确，同步时间命令

```bash
ntpdate ntp1.aliyun.com
```

2、token过期

主节点重新生成token及hash

```bash
kubeadm token create
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
```

```bash
kubeadm token create --print-join-command
kubeadm join 10.0.4.15:6443 --token eor7pr.fjduq50rq0jlbhit --discovery-token-ca-cert-hash sha256:95bba3d6fa9690dc7035dde37847b90c9ea17402eafd0e7937cf276fc17814ce
```

从节点重新执行kubeadm join（替换命令中的token及sha256）
```bash
kubeadm join 10.10.10.10:6443 --token aaaafd --discovery-token-ca-cert-hash sha256:1232asd
```

4. kubeadm join 异常： `/proc/sys/net/ipv4/ip_forward contents are not set to 1`
```bash
# kubeadm join 10.0.4.15:6443 --token eor7pr.fjduq50rq0jlbhit --discovery-token-ca-cert-hash sha256:95bba3d6fa9690dc7035dde37847b90c9ea17402eafd0e7937cf276fc17814ce --ignore-preflight-errors=...
[preflight] Running pre-flight checks
error execution phase preflight: [preflight] Some fatal errors occurred:
	[ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1
[preflight] If you know what you are doing, you can make a check non-fatal with `--ignore-preflight-errors=...`
To see the stack trace of this error execute with --v=5 or higher
```
解决办法：
`sysctl -w net.ipv4.ip_forward=1`



5. 启动 kube-flannel 异常：`pod cidr not assigned`
```bash
# 注意：每个worker节点的SUBNET需要区分开，否则k8s pods之间网络访问会不通。
kubectl patch node <NODE_NAME> -p '{"spec":{"podCIDR":"<SUBNET>"}}'


# 如下配置是init 时 cluster-cidr=172.18.0.0/16 所指定网段范围内的一个子网段
kubectl patch node node1 -p '{"spec":{"podCIDR":"172.18.1.0/24"}}'


kubectl describe node node1
````

6. coredns状态卡在ContainerCreating logs 显示: `is waiting to start: ContainerCreating`

步骤一：在所有节点（master和slave节点）删除cni0，以及暂停k8s和docker
```bash
kubeadm reset

systemctl stop kubelet
systemctl stop docker

rm -rf /var/lib/cni/
rm -rf /var/lib/kubelet/
rm -rf /etc/cni/

ifconfig cni0 down
ifconfig flannel.1 down
ifconfig docker0 down

ip link delete cni0
ip link delete flannel.1
```
 
步骤二：在所有节点重启kubelet和docker
```bash
systemctl start kubelet
systemctl start docker
```
 
步骤三：重新执行kubeadm init的操作

7. `kubeadm init` err: `[ERROR CRI]: container runtime is not running: output`
```bash
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
kubeadm init
```
8. 部署 calico 后一直处于 `CrashLoopBackOff` 状态,查看log 错误为：
```bash
2022-05-26 09:32:03.841 [INFO][9] startup/startup.go 483: Initialize BGP data
2022-05-26 09:32:03.843 [INFO][9] startup/autodetection_methods.go 103: Using autodetected IPv4 address on interface enp0s9: 10.0.4.15/24
2022-05-26 09:32:03.843 [INFO][9] startup/startup.go 559: Node IPv4 changed, will check for conflicts
2022-05-26 09:32:03.904 [WARNING][9] startup/startup.go 984: Calico node 'master' is already using the IPv4 address 10.0.4.15.
2022-05-26 09:32:03.904 [INFO][9] startup/startup.go 389: Clearing out-of-date IPv4 address from this node IP="10.0.4.15/24"
2022-05-26 09:32:03.945 [WARNING][9] startup/utils.go 49: Terminating
Calico node failed to start
```

或者

```bash
2022-05-26 09:48:14.774 [INFO][9] startup/startup.go 104: Datastore is ready
2022-05-26 09:48:14.826 [INFO][9] startup/startup.go 483: Initialize BGP data
2022-05-26 09:48:14.843 [WARNING][9] startup/autodetection_methods.go 140: Unable to auto-detect IPv4 address by connecting to DESTINATION: dial udp4: lookup DESTINATION on 192.168.124.1:53: no such host
2022-05-26 09:48:14.843 [WARNING][9] startup/startup.go 505: Couldn't autodetect an IPv4 address. If auto-detecting, choose a different autodetection method. Otherwise provide an explicit address.
2022-05-26 09:48:14.843 [INFO][9] startup/startup.go 389: Clearing out-of-date IPv4 address from this node IP=""
2022-05-26 09:48:14.880 [WARNING][9] startup/utils.go 49: Terminating
Calico node failed to start
```

解决办法：修改 calico.yaml 修改 `IP_AUTODETECTION_METHOD` 选项，并重新构建
```yaml
            - name: IP_AUTODETECTION_METHOD
          #    value: "interface=ens*"
              value: "interface=enp0s8"
          #    value: "can-reach=DESTINATION"
```

9. 删不掉 Terminating 状态的Pod
通过 `kubectl delete pods <pod-name> -n <ns-name>` 无法删除处于 Terminating 的pod

强制删除命令：`kubectl delete pods <pod-name> -n <ns-name> --grace-period=0 --force`

10. 












