---
title: kubernetes源码学习笔记
date: 2023-05-23 16:47:20
categories: [kubernetes, 源码]
tags:
- kubernetes
- private
---


## 代码分支：
    release-1.19

## 环境
- go version
```shell
  go version go1.17.3 linux/amd64
```
- kubectl version
```shell
  Client Version: version.Info{Major:"1", Minor:"22", GitVersion:"v1.22.4", GitCommit:"b695d79d4f967c403a96986f1750a35eb75e75f1", GitTreeState:"clean", BuildDate:"2021-11-17T15:48:33Z", GoVersion:"go1.16.10", Compiler:"gc", Platform:"linux/amd64"}
```
- uname -a
```shell
  Linux localhost.localhost 3.10.0-1160.45.1.el7.x86_64 #1 SMP Wed Oct 13 17:20:51 UTC 2021 x86_64 x86_64 x86_64 GNU/Linux
```
- docker version
```shell
  Client: Docker Engine - Community
  Version:           20.10.11
  API version:       1.41
  Go version:        go1.16.9
  Git commit:        dea9396
  Built:             Thu Nov 18 00:38:53 2021
  OS/Arch:           linux/amd64
  Context:           default
  Experimental:      true
```
- gcc --version
```shell
  gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44)
```



## Kubernetes 源码的目录结构：

| 目录名称      | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| api/          | 存放 OpenAPI/Swagger 的 spec 文件，包括 JSON、Protocol 的定义等 |
| build/        | 存放构建相关的脚本                                           |
| **cmd/**      | 存放可执行文件的**入口代码**，每一个可执行文件都会对应有一个 main 函数（**每个组件代码入口（main函数）**） |
| docs/         | 存放设计或者用户使用文档                                     |
| hack/         | 存放与构建、测试相关的脚本                                   |
| **pkg/**      | 存放核心库代码，可被项目内部或外部，直接引用                 |
| plugin/       | 存放 kubernetes 的插件，例如认证插件、授权插件等             |
| staging/      | 存放部分核心库的暂存代码，也就是还没有集成到 pkg 目录的代码  |
| **test/**     | 存放测试工具，以及测试数据                                   |
| third_party/  | 存放第三方工具、代码或其他组件                               |
| translations/ | 存放 i18n(国际化)语言包的相关文件，可以在不修改内部代码的情况下支持不同语言及地区 |
| **vendor/**   | 存放项目**依赖的库代码**，一般为第三方库代码                 |



## 整体架构

<img src="kubernetes架构.png" alt="kubernetes架构" style="zoom:100%;" />

## 分层架构

<img src="分层架构.png" alt="分层架构" style="zoom:100%;" />

## 模块划分：
- Kubernetes 主要由以下几个核心组件组成:
    - etcd 保存了整个集群的状态，就是一个数据库；
    - apiserver 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API 注册和发现等机制；
    - controller manager 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
    - scheduler 负责资源的调度，按照预定的调度策略将 Pod 调度到相应的机器上；
    - kubelet 负责维护容器的生命周期，同时也负责 Volume（CSI）和网络（CNI）的管理；
    - Container runtime 负责镜像管理以及 Pod 和容器的真正运行（CRI）；
    - kube-proxy 负责为 Service 提供 cluster 内部的服务发现和负载均衡；
    
- 除了上面的这些核心组件，还有一些推荐的插件：
    - kube-dns 负责为整个集群提供 DNS 服务
    - Ingress Controller 为服务提供外网入口
    - Heapster 提供资源监控
    - Dashboard 提供 GUI



### 核心资源概述:

Kubernetes 将资源再次分组和版本化 ， 形成 **Group (资源组〉、 Version (资源版本)、 Resource** **(资源)**。

- **Group** : 被称为**资源组**，在 Kubernetes API Server 中也可称其为 **APIGroup**.
- **Version** : 被称为**资源版本**，在 Kuberneles API Server 中也可称其为 **APIVersions** 。
- **Resourc****e**: 被称为**资源**，在 Kubernetes API Server 中也可称其为 **APIResource** 。
- **Kind**: **资源种类**，描述 Resource 的种类，**与 Resource 为同一级别**。



Kubemetes 系统支持多个 Group ，每个 Group 支持多个 Version， 每个 Version 支持多个 Resource ， 其中部分资源同时会拥有自己的子资源(即 SubResource) 。例如，Deployment 资源拥有 Status 子资源。

**组织形式**： 资源组、资源版本、资源、子资源的完整表现形式为： **<group>/<version>/<resource>/<subresource>**。以常用的 Deployment 资源为 例，其 完整表现形式为 apps/v1/deployment/status

另外 ， 资源对象 ( Resource Object ) ，由“资源组+资源版本+资源种类”组成 ， 并在实例化后表达一个资源对象 ，例如 Deployment 资源实例化后拥有资源组、资源版本及资源种类，其表现形式为 <group>/<version>，Kind=<kind> ，例如 apps/v1, Kind=Deployment.



可以通过 Group、Version、Resource 结构来标识一个资源的资源组名称、资源版本及资源名称。Group、Version、Resource 简称 **GVR**.



## 编译

### 安装依赖：
`docker、git、make`
`brew install gnu-tar`
`brew install coreutils`
### 下载源码：
- `cd $GOPATH/src/`
- `mkdir k8s.io && cd k8s.io`
- `git clone -v https://github.com/kubernetes/kubernetes`
- `cd kubernetes`
- `git checkout -b release-1.19 origin/release-1.19`

### 解决源码依赖：
- `go mod init`
- `go mod vendor`
> 一般版本直接进入下一步进行编译即可
### 源码编译：
- `make`、`make clean`
- 编译全部组件：`KUBE_BUILD_PLATFORMS=linux/amd64 make all GOFLAGS=-v GOGCFLAGS="-N -l"`
- 编译指定组件：`cd cmd/kube-apiserver && go build -v` 或 `KUBE_BUILD_PLATFORMS=linux/amd64 make WHAT=cmd/kube-apiserver GOFLAGS=-v GOGCFLAGS="-N -l"`
```shell
- KUBE_BUILD_PLATFORMS=linux/amd64 指定当前编译平台环境类型为 linux/amd64, mac 为 darwin/amd64
- make all 表示在本地环境中编译所有组件。
- GOFLAGS=-v 编译参数，开启 verbose 日志。
- GOGCFLAGS="-N -l" 编译参数，禁止编译优化和内联，减小可执行程序大小。 
```


### 异常处理：
1. 编译指定模块提示(`cd cmd/kube-apiserver && go build -v`)：
```shell
[root@localhost kube-apiserver]# go build .
go: updates to go.mod needed, disabled by -mod=vendor
	(Go version in go.mod is at least 1.14 and vendor directory exists.)
	to update it:
	go mod tidy
go: updates to go.mod needed, disabled by -mod=vendor
	(Go version in go.mod is at least 1.14 and vendor directory exists.)
	to update it:
	go mod tidy
```
解决办法：
```shell
go get -u
go mod tidy
go mod vendor
```
2. `make all` 或 `KUBE_BUILD_PLATFORMS=linux/amd64 make all GOFLAGS=-v GOGCFLAGS="-N -l"`提示异常：
```shell
...
	To ignore the vendor directory, use -mod=readonly or -mod=mod.
	To sync the vendor directory, run:
		go mod vendor
!!! [1028 15:40:37] Call tree:
!!! [1028 15:40:37]  1: /home/go_worker/src/k8s.io/kubernetes/hack/lib/golang.sh:616 kube::golang::build_some_binaries(...)
!!! [1028 15:40:37]  2: /home/go_worker/src/k8s.io/kubernetes/hack/lib/golang.sh:752 kube::golang::build_binaries_for_platform(...)
!!! [1028 15:40:37]  3: hack/make-rules/build.sh:27 kube::golang::build_binaries(...)
!!! [1028 15:40:37] Call tree:
!!! [1028 15:40:37]  1: hack/make-rules/build.sh:27 kube::golang::build_binaries(...)
!!! [1028 15:40:37] Call tree:
!!! [1028 15:40:37]  1: hack/make-rules/build.sh:27 kube::golang::build_binaries(...)
make[1]: *** [_output/bin/deepcopy-gen] 错误 1
make: *** [generated_files] 错误 2
```
解决办法：
此问题出现在1.13版本，可能是因为go mod得问题，建议使用Godeps

### 编译结果：
``` shell
[root@localhost kubernetes]# ll _output/
总用量 80
-rw-r--r-- 1 root root  3378 10月 28 16:00 AGGREGATOR_violations.report
-rw-r--r-- 1 root root  3378 10月 28 16:00 APIEXTENSIONS_violations.report
lrwxrwxrwx 1 root root    67 10月 28 16:09 bin -> /home/go_worker/src/k8s.io/kubernetes/_output/local/bin/linux/amd64
-rw-r--r-- 1 root root  3713 10月 28 16:00 CODEGEN_violations.report
-rw-r--r-- 1 root root 64197 10月 28 16:00 KUBE_violations.report
drwxr-xr-x 4 root root    27 10月 28 15:59 local
-rw-r--r-- 1 root root  3492 10月 28 16:00 SAMPLEAPISERVER_violations.report
```

### 清除：
`make clean`

### 运行:
```shell
  make clean && KUBE_BUILD_PLATFORMS=linux/amd64 make all GOFLAGS=-v GOGCFLAGS="-N -l"
```

## 官方文档链接
https://kubernetes.io/zh/docs/home/


## 模块学习

### apiserver

#### 主要功能：
Kubernetes API 服务器验证并配置 API 对象的数据， 这些对象包括 pods、services、replicationcontrollers 等。 API 服务器为 REST 操作提供服务，并为集群的共享状态提供前端， 所有其他组件都通过该前端进行交互.

API Server提供了k8s各类资源对象（pod,RC,Service等）的增删改查及watch等HTTP Rest接口，是整个系统的数据总线和数据中心。

- 提供了集群管理的REST API接口(包括认证授权、数据校验以及集群状态变更)；
- 提供其他模块之间的数据交互和通信的枢纽（其他模块通过API Server查询或修改数据，只有API Server才直接操作etcd）;
- 是资源配额控制的入口；
- 拥有完备的集群安全机制.

<img src="kube-apiserver.png" alt="apiserver" style="zoom:50%;" />


#### 使用：
`kube-apiserver [flags]`

##### 主要参数:
- --admission-control-config-file string: 包含准入控制配置的文件
- --advertise-address string: 向集群成员通知 apiserver 消息的 IP 地址。 这个地址必须能够被集群中其他成员访问。 如果 IP 地址为空，将会使用 --bind-address， 如果未指定 --bind-address，将会使用主机的默认接口地址
- --alsologtostderr: 在向文件输出日志的同时，也将日志写到标准输出
- --apiserver-count int(默认值：1): 集群中运行的 API 服务器数量，必须为正数。 （在启用 `--endpoint-reconciler-type=master-count` 时使用。）
- --audit-log-compress: 若设置了此标志，则被轮换的日志文件会使用 gzip 压缩
- --audit-log-maxage int: 根据文件名中编码的时间戳保留旧审计日志文件的最大天数
- --audit-policy-file string: 定义审计策略配置的文件的路径
- 更多选项参考： [官方文档apiserver部分](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-apiserver/)

#### 运行
在单个k8s-master节点上。默认有两个端口

##### 1. 本地端口
- 该端口用于接收HTTP请求；
- 该端口默认值为8080，可以通过API Server的启动参数“--insecure-port”的值来修改默认值；
- 默认的IP地址为“localhost”，可以通过启动参数“--insecure-bind-address”的值来修改该IP地址；
- 非认证或授权的HTTP请求通过该端口访问API Server。

##### 2. 安全端口
- 该端口默认值为6443，可通过启动参数“--secure-port”的值来修改默认值；
- 默认IP地址为非本地（Non-Localhost）网络端口，通过启动参数“--bind-address”设置该值；
- 该端口用于接收HTTPS请求；
- 用于基于Tocken文件或客户端证书及HTTP Base的认证；
- 用于基于策略的授权；
- 默认不启动HTTPS安全访问控制。

#### 代码入口：
- `cmd/kube-apiserver/apiserver.go -> main()`: main() 函数入口位置,
- `cmd/kube-apiserver/app/server.go -> NewAPIServerCommand()`
-

#### 业务逻辑：

##### 延申：

### scheduler(调度器)

#### 主要功能：
Scheduler是一个跑在其他组件边上的独立程序，对接Apiserver寻找PodSpec.NodeName为空的Pod，然后用post的方式发送一个api调用，指定这些pod应该跑在哪个node上。

通俗地说，就是scheduler是相对独立的一个组件，主动访问api server，寻找等待调度的pod，然后通过一系列调度算法寻找哪个node适合跑这个pod，然后将这个pod和node的绑定关系发给api server，从而完成了调度的过程。

调度器基于约束和可用资源为调度队列中每个 Pod 确定其可合法放置的节点。 调度器之后对所有合法的节点进行排序，将 Pod 绑定到一个合适的节点。 在同一个集群中可以使用多个不同的调度器；

Scheduler负责Pod调度。在整个系统中起"承上启下"作用，承上：负责接收Controller Manager创建的新的Pod，为其选择一个合适的Node；启下：Node上的kubelet接管Pod的生命周期。

特殊的 Controller，工作原理与其他控制器无差别。

Scheduler 的特殊职责在于监控当前集群所有未调度的 Pod，并且获取当前集群所有节点的健康状况和资源使用情况，为待调度 Pod 选择最佳计算节点，完成调度。

调度阶段分为：
  - Predict：过滤不能满足业务需求的节点，如资源不足、端口冲突等。 
  - Priority：按既定要素将满足调度需求的节点评分，选择最佳节点。 
  - Bind：将计算节点与 Pod 绑定，完成调度。

流程:

<img src="Scheduler.png" alt="Scheduler" style="zoom:100%;" />


#### 使用：
`kube-scheduler [flags]`

默认调度策略是通过`defaultPredicates()` 和 `defaultPriorities()函数`定义的， 源码在 `pkg/scheduler/algorithmprovider/defaults/defaults.go`，可以通过命令行flag `--policy-config-file`来覆盖默认行为。所以可以通过配置文件的方式或者修改`pkg/scheduler/algorithm/predicates/predicates.go` /`pkg/scheduler/algorithm/priorities`，然后注册到`defaultPredicates()`/`defaultPriorities()`来实现。配置文件类似下面这个样子：

```yaml
{
"apiVersion" : "v1",
"kind" : "Policy",
"predicates" : [
    {"name" : "PodFitsHostPorts"},
    {"name" : "PodFitsResources"},
    {"name" : "NoDiskConflict"},
    {"name" : "NoVolumeZoneConflict"},
    {"name" : "MatchNodeSelector"},
    {"name" : "HostName"}
    ],
"priorities" : [
    {"name" : "LeastRequestedPriority", "weight" : 1},
    {"name" : "BalancedResourceAllocation", "weight" : 1},
    {"name" : "ServiceSpreadingPriority", "weight" : 1},
    {"name" : "EqualPriority", "weight" : 1}
    ],
"hardPodAffinitySymmetricWeight" : 10,
"alwaysCheckAllPredicates" : false
}
```

#### 主要参数：
- --config string: 配置文件的路径。以下标志会覆盖此文件中的值：
  --algorithm-provider
  --policy-config-file
  --policy-configmap
  --policy-configmap-namespace
- --leader-elect (默认值：true): 在执行主循环之前，开始领导者选举并选出领导者。 使用多副本来实现高可用性时，可启用此标志
- 更多选项参考： [官方文档scheduler部分](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kube-scheduler/)


#### 预选策略
说明：返回true表示该节点满足该Pod的调度条件；返回false表示该节点不满足该Pod的调度条件。

##### 1. NoDiskConflict
判断备选Pod的数据卷是否与该Node上已存在Pod挂载的数据卷冲突，如果是则返回false，否则返回true。

##### 2. PodFitsResources
判断备选节点的资源是否满足备选Pod的需求，即节点的剩余资源满不满足该Pod的资源使用。

- 计算备选Pod和节点中已用资源（该节点所有Pod的使用资源）的总和。
- 获取备选节点的状态信息，包括节点资源信息。
- 如果（备选Pod+节点已用资源>该节点总资源）则返回false，即剩余资源不满足该Pod使用；否则返回true。

##### 3. PodSelectorMatches
判断节点是否包含备选Pod的标签选择器指定的标签，即通过标签来选择Node。

- 如果Pod中没有指定spec.nodeSelector，则返回true。
- 否则获得备选节点的标签信息，判断该节点的标签信息中是否包含该Pod的spec.nodeSelector中指定的标签，如果包含返回true，否则返回false。


##### 4. PodFitsHost
判断备选Pod的spec.nodeName所指定的节点名称与备选节点名称是否一致，如果一致返回true，否则返回false。

##### 5. CheckNodeLabelPresence
检查备选节点中是否有Scheduler配置的标签，如果有返回true，否则返回false。


##### 6. CheckServiceAffinity
判断备选节点是否包含Scheduler配置的标签，如果有返回true，否则返回false。

##### 7. PodFitsPorts
判断备选Pod所用的端口列表中的端口是否在备选节点中已被占用，如果被占用返回false，否则返回true。

#### 优选策略

##### 1. LeastRequestedPriority
优先从备选节点列表中选择资源消耗最小的节点（CPU+内存）。

##### 2. CalculateNodeLabelPriority
优先选择含有指定Label的节点。

###### 3. BalancedResourceAllocation
优先从备选节点列表中选择各项资源使用率最均衡的节点。


#### Pod 创建流程
<img src="what-happens-when-k8s.svg" alt="pod 创建流程" style="zoom:50%;" />


#### 代码入口：
- `cmd/kube-scheduler/scheduler.go`: main() 函数入口位置，在scheduler过程开始被调用前的一系列初始化工作。
- `pkg/scheduler/scheduler.go`: 调度框架的整体逻辑，在具体的调度算法之上的框架性的代码。
- `pkg/scheduler/core/generic_scheduler.go`: 具体的计算哪些node适合跑哪些pod的算法。

#### 业务逻辑：
```shell
对于一个给定的pod
+---------------------------------------------+
|             可用于调度的nodes如下：           |
|  +--------+     +--------+     +--------+   |
|  | node 1 |     | node 2 |     | node 3 |   |
|  +--------+     +--------+     +--------+   |
+----------------------+----------------------+
                       |
                       v
+----------------------+----------------------+
初步过滤: node 3 资源不足
+----------------------+----------------------+
                       |
                       v
+----------------------+----------------------+
|                 剩下的nodes:                 |
|     +--------+               +--------+     |
|     | node 1 |               | node 2 |     |
|     +--------+               +--------+     |
+----------------------+----------------------+
                       |
                       v
+----------------------+----------------------+
优先级算法计算结果:    node 1: 分数=2
                    node 2: 分数=5
+----------------------+----------------------+
                       |
                       v
            选择分值最高的节点 = node 2
```

Scheduler为每个pod寻找一个适合其运行的node，大体分成三步：

1. 通过一系列的“predicates”过滤掉不能运行pod的node，比如一个pod需要500M的内存，有些节点剩余内存只有100M了，就会被剔除；
2. 通过一系列的“priority functions”给剩下的node排一个等级，分出三六九等，寻找能够运行pod的若干node中最合适的一个node；
3. 得分最高的一个node，也就是被“priority functions”选中的node胜出了，获得了跑对应pod的资格。





##### 延申：

- Predicates是一些用于过滤不合适node的策略。

- Priorities是一些用于区分node排名（分数）的策略（作用在通过predicates过滤的node上）。

  Predicates 和 priorities 的代码分别在：

  - pkg/scheduler/algorithm/predicates/predicates.go
  - pkg/scheduler/algorithm/priorities.




### controller(控制器)

#### 1. controller Manager 

##### 主要功能：

Controller Manager作为集群内部的管理控制中心，负责集群内的Node、Pod副本、服务端点（Endpoint）、命名空间（Namespace）、服务账号（ServiceAccount）、资源定额（ResourceQuota）的管理，当某个Node意外宕机时，Controller Manager会及时发现并执行自动化修复流程，确保集群始终处于预期的工作状态。

每个Controller通过API Server提供的接口实时监控整个集群的每个资源对象的当前状态，当发生各种故障导致系统状态发生变化时，会尝试将系统状态修复到“期望状态”。

- Controller Manager 是集群的大脑，是确保整个集群动起来的关键；
- 作用是确保 Kubernetes 遵循声明式系统规范，确保系统的真实状态（Actual State）与用户定义的期望状态（Desired State）一致；
- Controller Manager 是多个控制器的组合，每个 Controller 事实上都是一个control loop，负责侦听其管控的对象，当对象发生变更时完成配置；
- Controller 配置失败通常会触发自动重试，整个集群会在控制器不断重试的机制下确保最终一致性（ Eventual Consistency）.

<img src="controller-manager.png" alt="controller" style="zoom:50%;" />


##### 使用：

##### 主要参数：

##### 代码入口：

##### 业务逻辑：

##### 延申：


#### 2. Replication Controller



#### 3. Node Controller


#### 4. ResourceQuota Controller

#### 5. Namespace Controller

#### 6. Endpoint Controller



#### 7. Service Controller
Service Controller是属于kubernetes集群与外部的云平台之间的一个接口控制器。Service Controller监听Service变化，如果是一个LoadBalancer类型的Service，则确保外部的云平台上对该Service对应的LoadBalancer实例被相应地创建、删除及更新路由转发表。


### kubelet

#### 主要功能：
kubelet 是在每个 Node 节点上运行的主要 “节点代理”。它可以使用以下之一向 apiserver 注册： 主机名（hostname）；覆盖主机名的参数；某云驱动的特定逻辑。

kubelet 是基于 PodSpec 来工作的。每个 PodSpec 是一个描述 Pod 的 YAML 或 JSON 对象。 kubelet 接受通过各种机制（主要是通过 apiserver）提供的一组 PodSpec，并确保这些 PodSpec 中描述的容器处于运行状态且运行状况良好。 kubelet 不管理不是由 Kubernetes 创建的容器。

除了来自 apiserver 的 PodSpec 之外，还可以通过以下三种方式将容器清单（manifest）提供给 kubelet。

- 文件（File）：利用命令行参数传递路径。kubelet 周期性地监视此路径下的文件是否有更新。 监视周期默认为 20s，且可通过参数进行配置。
- HTTP 端点（HTTP endpoint）：利用命令行参数指定 HTTP 端点。 此端点的监视周期默认为 20 秒，也可以使用参数进行配置。
- HTTP 服务器（HTTP server）：kubelet 还可以侦听 HTTP 并响应简单的 API （目前没有完整规范）来提交新的清单。

Kubernetes 的初始化系统（init system）
- 从不同源获取 Pod 清单，并按需求启停 Pod 的核心组件： 
- Pod 清单可从本地文件目录，给定的 HTTPServer 或 Kube-APIServer 等源头获取； 
- Kubelet 将运行时，网络和存储抽象成了 CRI，CNI，CSI。 
- 负责汇报当前节点的资源信息和健康状态； 
- 负责 Pod 的健康检查和状态汇报。

流程:

<img src="Kubelet.png" alt="Kubelet" style="zoom:100%;" />

#### 使用：
`kubelet [flags]`

#### 主要参数：
- --address ip(默认值：0.0.0.0) : kubelet 用来提供服务的 IP 地址（设置为0.0.0.0 表示使用所有 IPv4 接口， 设置为 :: 表示使用所有 IPv6 接口）
- --anonymous-auth (默认值：true) : 设置为 true 表示 kubelet 服务器可以接受匿名请求。未被任何认证组件拒绝的请求将被视为匿名请求。 匿名请求的用户名为 system:anonymous，用户组为 system:unauthenticated
- --authentication-token-webhook : 使用 TokenReview API 对持有者令牌进行身份认证
- --authorization-mode string : kubelet 服务器的鉴权模式。可选值包括：AlwaysAllow、Webhook。Webhook 模式使用 SubjectAccessReview API 鉴权。 当 --config 参数未被设置时，默认值为 AlwaysAllow，当使用了 --config 时，默认值为 Webhook
- --bootstrap-kubeconfig string : 某 kubeconfig 文件的路径，该文件将用于获取 kubelet 的客户端证书。 如果 --kubeconfig 所指定的文件不存在，则使用引导所用 kubeconfig 从 API 服务器请求客户端证书。成功后，将引用生成的客户端证书和密钥的 kubeconfig 写入 --kubeconfig 所指定的路径。客户端证书和密钥文件将存储在 --cert-dir 所指的目录
- --cert-dir string (默认值：/var/lib/kubelet/pki) : TLS 证书所在的目录。如果设置了 --tls-cert-file 和 --tls-private-key-file， 则此标志将被忽略
- --cluster-dns strings: DNS 服务器的 IP 地址，以逗号分隔。此标志值用于 Pod 中设置了 “dnsPolicy=ClusterFirst” 时为容器提供 DNS 服务
- --cluster-domain string: 集群的域名。如果设置了此值，kubelet 除了将主机的搜索域配置到所有容器之外，还会为其 配置所搜这里指定的域名。 --config 时，默认值为 Webhook
- 更多选项参考： [官方文档kubelet部分](https://kubernetes.io/zh/docs/reference/command-line-tools-reference/kubelet/)

#### 代码入口：

#### 业务逻辑：

##### 延申：


### kube-proxy

#### 主要功能：

- 监控集群中用户发布的服务，并完成负载均衡配置。
- 每个节点的 Kube-Proxy 都会配置相同的负载均衡策略，使得整个集群的服务发现建立在分布式负载均衡器之上，服务调用无需经过额外的网络跳转（Network Hop）。
- 负载均衡配置基于不同插件实现：
  - userspace。
  - 操作系统网络协议栈不同的 Hooks 点和插件： 
    - iptables； 
    - ipvs。

流程:

<img src="Kube-Proxy.png" alt="Kube-Proxy" style="zoom:100%;" />


#### 使用：

#### 主要参数：

#### 代码入口：

#### 业务逻辑：

##### 延申：


### client-go

#### Ingress Controller

##### 主要功能：

##### 使用：

##### 主要参数：

##### 代码入口：

##### 业务逻辑：

##### 延申：

#### Informer

##### 主要功能：

是Kubernetes中的基础组件，负责各组件与Apiserver的资源与事件同步

##### 使用：

##### 主要参数：

##### 代码入口：

##### 业务逻辑：

##### 延申：

#### factory 

##### 主要功能：

##### 使用：

##### 主要参数：

##### 代码入口：

##### 业务逻辑：

##### 延申：