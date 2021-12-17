# 安装部署手册
 中文 容器管理平台 简称Ksc

从 v3.0.0 开始，[Ksc]将基于 ansible 的安装程序更改为使用 Go 语言开发的名为 KubeKey 的新安装程序。使用 KubeKey，您可以轻松、高效、灵活地单独或整体安装 Kubernetes 和 Ksc。


## 支持的环境

### Linux 发行版

* **Ubuntu**  *16.04, 18.04, 20.04*
* **Debian**  *Buster, Stretch*
* **CentOS/RHEL**  *7*
* **SUSE Linux Enterprise Server** *15*

> 建议使用 Linux Kernel 版本: `4.15 or later` \
> 可以通过命令 `uname -srm` 查看 Linux Kernel 版本。

###Kubernetes 版本

v1.17.9*
v1.18.6*
v1.19.8*  
v1.20.6*
v1.21.5*  (default)默认安装版本
v1.22.1*


## 要求和建议

* 最低资源要求（仅对于最小安装 KubeSphere）：
  * 2 核虚拟 CPU
  * 4 GB 内存
  * 20 GB 储存空间

/var/lib/docker 主要用于存储容器数据，在使用和操作过程中会逐渐增大。对于生产环境，建议/var/lib/docker 单独挂盘。

###mount /dev/sdb /var/lib/docker 需加入到fatab中实现开机自启

* 操作系统要求：
  * `SSH` 可以访问所有节点。
  * 所有节点的时间同步。
  * `sudo`/`curl`/`openssl` 应在所有节点使用。
  * `docker` 可以自己安装，也可以通过 KubeKey 安装。
  * `Red Hat` 在其 `Linux` 发行版本中包括了`SELinux`，建议[关闭SELinux]

> * 建议您的操作系统环境足够干净 (不安装任何其他软件)，否则可能会发生冲突。
> * 如果在从 dockerhub.io 下载镜像时遇到问题，建议准备一个容器镜像仓库 (加速器)。[为 Docker 守护程序配置镜像加速](https://docs.docker.com/registry/recipes/mirror/#configure-the-docker-daemon)。
> * 对于生产，请使用 NFS/Ceph/GlusterFS 或商业化存储作为持久化存储，并在所有节点中安装[相关的客户端]
> * 如果遇到拷贝时报权限问题Permission denied,建议优先考虑查看[SELinux的原因]

* 依赖要求:

KubeKey 可以同时安装 Kubernetes 和 Ksc。根据 Ksc 所安装版本的不同，您所需要安装的依赖可能也不同。请参考以下表格查看您是否需要提前在节点上安装有关的依赖。

|             | Kubernetes 版本 ≥ 1.18 | Kubernetes 版本 < 1.18 |
| ----------- | ---------------------- | ---------------------- |
| `socat`     | 必须安装               | 可选，但推荐安装       |
| `conntrack` | 必须安装               | 可选，但推荐安装       |
| `ebtables`  | 可选，但推荐安装       | 可选，但推荐安装       |
| `ipset`     | 可选，但推荐安装       | 可选，但推荐安装       |

* 网络和 DNS 要求：
  * 确保 `/etc/resolv.conf` 中的 DNS 地址可用。否则，可能会导致集群中出现某些 DNS 问题。
  * 如果您的网络配置使用防火墙或安全组，则必须确保基础结构组件可以通过特定端口相互通信。建议您关闭防火墙或遵循链接配置：[网络访问]。

## 用法

### 获取安装程序可执行文件

* 下载KubeKey可执行文件kk 


### 创建集群

#### 快速开始

请先执行 `export KKZONE=cn`.

1. 首先，配置文件如下config-sample.yaml
```
apiVersion: kubekey.kubesphere.io/v1alpha1
kind: Cluster
metadata:
  name: sample
spec:
  hosts:
  - {name: master, address: 192.168.4.31, internalAddress: 192.168.4.31, user: root, password: Xl123456..}
  - {name: node1, address: 192.168.4.32, internalAddress: 192.168.4.32, user: root, password: Xl123456..}
  - {name: node2, address: 192.168.4.33, internalAddress: 192.168.4.33, user: root, password: Xl123456..}
  roleGroups:
    etcd:
    - master
    master: 
    - master
    worker:
    - node1
    - node2
  controlPlaneEndpoint:
    domain: lb.kubesphere.local
    address: ""
    port: 6443
  kubernetes:
    version: v1.20.4
    imageRepo: kubesphere
    clusterName: cluster.local
  network:
    plugin: calico
    kubePodsCIDR: 10.233.64.0/18
    kubeServiceCIDR: 10.233.0.0/18
  registry:
    registryMirrors: []
    insecureRegistries: []
  addons: 
  - name: nfs-client
    namespace: kube-system
    sources:
      chart:
        name: nfs-client-provisioner
        repo: https://charts.kubesphere.io/main
        valuesFile: /root/nfs-client.yaml 


---
apiVersion: installer.kubesphere.io/v1alpha1
kind: ClusterConfiguration
metadata:
  name: ks-installer
  namespace: kubesphere-system
  labels:
    version: v3.1.0
spec:
  persistence:
    storageClass: ""       
  authentication:
    jwtSecret: ""
  zone: ""
  local_registry: ""        
  etcd:
    monitoring: false      
    endpointIps: localhost  
    port: 2379             
    tlsEnable: true
  common:
    redis:
      enabled: false
    redisVolumSize: 2Gi 
    openldap:
      enabled: false
    openldapVolumeSize: 2Gi  
    minioVolumeSize: 20Gi
    monitoring:
      endpoint: http://prometheus-operated.kubesphere-monitoring-system.svc:9090
    es:  
      elasticsearchMasterVolumeSize: 4Gi   
      elasticsearchDataVolumeSize: 20Gi   
      logMaxAge: 7          
      elkPrefix: logstash
      basicAuth:
        enabled: false
        username: ""
        password: ""
      externalElasticsearchUrl: ""
      externalElasticsearchPort: ""  
  console:
    enableMultiLogin: true 
    port: 30880
  alerting:       
    enabled: false
    # thanosruler:
    #   replicas: 1
    #   resources: {}
  auditing:    
    enabled: false
  devops:           
    enabled: false
    jenkinsMemoryLim: 2Gi     
    jenkinsMemoryReq: 1500Mi 
    jenkinsVolumeSize: 8Gi   
    jenkinsJavaOpts_Xms: 512m  
    jenkinsJavaOpts_Xmx: 512m
    jenkinsJavaOpts_MaxRAM: 2g
  events:          
    enabled: false
    ruler:
      enabled: true
      replicas: 2
  logging:         
    enabled: false
    logsidecar:
      enabled: true
      replicas: 2
  metrics_server:             
    enabled: false
  monitoring:
    storageClass: ""
    prometheusMemoryRequest: 400Mi  
    prometheusVolumeSize: 20Gi  
  multicluster:
    clusterRole: none 
  network:
    networkpolicy:
      enabled: false
    ippool:
      type: none
    topology:
      type: none
  openpitrix:
    store:
      enabled: false
  servicemesh:    
    enabled: false  
  kubeedge:
    enabled: false
    cloudCore:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      cloudhubPort: "10000"
      cloudhubQuicPort: "10001"
      cloudhubHttpsPort: "10002"
      cloudstreamPort: "10003"
      tunnelPort: "10004"
      cloudHub:
        advertiseAddress: 
          - ""           
        nodeLimit: "100"
      service:
        cloudhubNodePort: "30000"
        cloudhubQuicNodePort: "30001"
        cloudhubHttpsNodePort: "30002"
        cloudstreamNodePort: "30003"
        tunnelNodePort: "30004"
    edgeWatcher:
      nodeSelector: {"node-role.kubernetes.io/worker": ""}
      tolerations: []
      edgeWatcherAgent:
        nodeSelector: {"node-role.kubernetes.io/worker": ""}
        tolerations: []
```
主机
请参照上方示例在 hosts 下列出您的所有机器并添加详细信息。

name：实例的主机名。

address：任务机和其他实例通过 SSH 相互连接所使用的 IP 地址。根据您的环境，可以是公有 IP 地址或私有 IP 地址。例如，一些云平台为每个实例提供一个公有 IP 地址，用于通过 SSH 访问。在这种情况下，您可以在该字段填入这个公有 IP 地址。

internalAddress：实例的私有 IP 地址。

此外，您必须提供用于连接至每台实例的登录信息，以下示例供您参考：

使用密码登录示例：

hosts:
  - {name: master, address: 192.168.0.2, internalAddress: 192.168.0.2, port: 8022, user: test, password: Testing123}
备注

在本教程中，端口 22 是 SSH 的默认端口，因此您无需将它添加至该 YAML 文件中。否则，您需要在 IP 地址后添加对应端口号，如上所示。

默认 root 用户示例：

hosts:
  - {name: master, address: 192.168.0.2, internalAddress: 192.168.0.2, password: Testing123}
使用 SSH 密钥的无密码登录示例：

hosts:
  - {name: master, address: 192.168.0.2, internalAddress: 192.168.0.2, privateKeyPath: "~/.ssh/id_rsa"}
提示

在安装 Ksc之前，您可以使用 hosts 下提供的信息（例如 IP 地址和密码）通过 SSH 的方式测试任务机和其他实例之间的网络连接。
在安装前，请确保端口 6443 没有被其他服务占用，否则在安装时会产生冲突（6443 为 API 服务器的默认端口）。
roleGroups
etcd：etcd 节点名称
master：主节点名称
worker：工作节点名称
controlPlaneEndpoint（仅适用于高可用安装）
您需要在 controlPlaneEndpoint 部分为高可用集群提供外部负载均衡器信息。当且仅当您安装多个主节点时，才需要准备和配置外部负载均衡器。请注意，config-sample.yaml 中的地址和端口应缩进两个空格，address 应为您的负载均衡器地址。有关详细信息，请参见[高可用配置]。

addons
您可以在 config-sample.yaml 的 addons 字段下指定存储，从而自定义持久化存储插件，例如 NFS 客户端、Ceph RBD、GlusterFS 等。有关更多信息，请参见[持久化存储配置] 生产环境必配置项。

Ksc 会默认安装 OpenEBS，为开发和测试环境配置 LocalPV，方便新用户使用。在本多节点安装示例中，使用了默认存储类型（本地存储卷）。对于生产环境，您可以使用 NFS/Ceph/GlusterFS/CSI 或者商业存储产品作为持久化存储解决方案。


2. 根据您的环境修改配置文件 config-sample.yaml
> 注意： 由于 Kubernetes 暂不支持大写 NodeName， worker 节点名中包含大写字母将导致后续安装过程无法正常结束
>
> 当指定安装KubeSphere时，要求集群中有可用的持久化存储。默认使用localVolume，如果需要使用其他持久化存储，请参阅 [addons]配置。
3. 使用配置文件创建集群。

      ```shell script
      ./kk create cluster -f config-sample.yaml
      ```


### 部署ksc 前端文件
kubectl apply -f configmap.yaml 
kubectl apply -f deploy.yaml
kubectl apply -f server.yaml
访问http://[nodeip]:30882 


### 添加节点

将新节点的信息添加到集群配置文件，然后应用更改。

```shell script
./kk add nodes -f config-sample.yaml
```
### 删除节点

通过以下命令删除节点，nodename指需要删除的节点名。

```shell script
./kk delete node <nodeName> -f config-sample.yaml
```

### 删除集群

您可以通过以下命令删除集群：

* 如果您以快速入门（all-in-one）开始：

```shell script
./kk delete cluster
```

* 如果从高级安装开始（使用配置文件创建的集群）：

```shell script
./kk delete cluster [-f config-sample.yaml]
```



### 启用 kubectl 自动补全

KubeKey 不会启用 kubectl 自动补全功能。请参阅下面的指南并将其打开：

**先决条件**：确保已安装 `bash-autocompletion` 并可以正常工作。

```shell script
# 安装 bash-completion
apt-get install bash-completion

# 将 completion 脚本添加到你的 ~/.bashrc 文件
echo 'source <(kubectl completion bash)' >>~/.bashrc

# 将 completion 脚本添加到 /etc/bash_completion.d 目录
kubectl completion bash >/etc/bash_completion.d/kubectl
```


### 集群升级

#### 多节点集群
通过指定配置文件对集群进行升级。
```shell script
./kk upgrade [--with-kubernetes version] [--with-ksc version] [(-f | --file) path]
```
* `--with-kubernetes` 指定kubernetes目标版本。
* `--with-ksc` 指定kubesphere目标版本。
* `-f` 指定集群安装时创建的配置文件。

> 注意: 升级多节点集群需要指定配置文件. 如果集群非kubekey创建，或者创建集群时生成的配置文件丢失，需要重新生成配置文件，或使用以下方法生成。

Getting cluster info and generating kubekey's configuration file (optional).
```shell script
./kk create config [--from-cluster] [(-f | --file) path] [--kubeconfig path]
```
* `--from-cluster` 根据已存在集群信息生成配置文件. 
* `-f` 指定生成配置文件路径.
* `--kubeconfig` 指定集群kubeconfig文件. 
* 由于无法全面获取集群配置，生成配置文件后，请根据集群实际信息补全配置文件。


### 高可用
使用 Keepalived 和 HAproxy 创建高可用集群
高可用 Kubernetes 集群能够确保应用程序在运行时不会出现服务中断，这也是生产的需求之一。为此，有很多方法可供选择以实现高可用。

本教程演示了如何配置 Keepalived 和 HAproxy 使负载均衡、实现高可用。步骤如下：

准备主机。
配置 Keepalived 和 HAproxy。
使用 KubeKey 创建 Kubernetes 集群，并安装 Ksc。
集群架构
示例集群有三个主节点，三个工作节点，两个用于负载均衡的节点，以及一个虚拟 IP 地址。本示例中的虚拟 IP 地址也可称为“浮动 IP 地址”。这意味着在节点故障的情况下，该 IP 地址可在节点之间漂移，从而实现高可用。

architecture-ha-k8s-cluster

请注意，在本示例中，Keepalived 和 HAproxy 没有安装在任何主节点上。但您也可以这样做，并同时实现高可用。然而，配置两个用于负载均衡的特定节点（您可以按需增加更多此类节点）会更加安全。这两个节点上只安装 Keepalived 和 HAproxy，以避免与任何 Kubernetes 组件和服务的潜在冲突。

准备主机
IP 地址	主机名	角色
172.16.0.2	lb1	Keepalived & HAproxy
172.16.0.3	lb2	Keepalived & HAproxy
172.16.0.4	master1	master, etcd
172.16.0.5	master2	master, etcd
172.16.0.6	master3	master, etcd
172.16.0.7	worker1	worker
172.16.0.8	worker2	worker
172.16.0.9	worker3	worker
172.16.0.10		虚拟 IP 地址
有关更多节点、网络、依赖项等要求的信息，请参见多节点安装。

配置负载均衡
Keepalived 提供 VRPP 实现，并允许您配置 Linux 机器使负载均衡，预防单点故障。HAProxy 提供可靠、高性能的负载均衡，能与 Keepalived 完美配合。

由于 lb1 和 lb2 上安装了 Keepalived 和 HAproxy，如果其中一个节点故障，虚拟 IP 地址（即浮动 IP 地址）将自动与另一个节点关联，使群集仍然可以正常运行，从而实现高可用。若有需要，也可以此为目的，添加更多安装 Keepalived 和 HAproxy 的节点。

先运行以下命令安装 Keepalived 和 HAproxy。

yum install keepalived haproxy psmisc -y
HAproxy
在两台用于负载均衡的机器上运行以下命令以配置 Proxy（两台机器的 Proxy 配置相同）：

vi /etc/haproxy/haproxy.cfg
以下是示例配置，供您参考（请注意 server 字段。请记住 6443 是 apiserver 端口）：

global
    log /dev/log  local0 warning
    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon
   stats socket /var/lib/haproxy/stats
defaults
  log global
  option  httplog
  option  dontlognull
        timeout connect 5000
        timeout client 50000
        timeout server 50000
frontend kube-apiserver
  bind *:6443
  mode tcp
  option tcplog
  default_backend kube-apiserver
backend kube-apiserver
    mode tcp
    option tcplog
    option tcp-check
    balance roundrobin
    default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
    server kube-apiserver-1 172.16.0.4:6443 check # Replace the IP address with your own.
    server kube-apiserver-2 172.16.0.5:6443 check # Replace the IP address with your own.
    server kube-apiserver-3 172.16.0.6:6443 check # Replace the IP address with your own.
保存文件并运行以下命令以重启 HAproxy。

systemctl restart haproxy
使 HAproxy 在开机后自动运行：

systemctl enable haproxy
确保您在另一台机器 (lb2) 上也配置了 HAproxy。

Keepalived
两台机器上必须都安装 Keepalived，但在配置上略有不同。

运行以下命令以配置 Keepalived。

vi /etc/keepalived/keepalived.conf
以下是示例配置 (lb1)，供您参考：

global_defs {
  notification_email {
  }
  router_id LVS_DEVEL
  vrrp_skip_check_adv_addr
  vrrp_garp_interval 0
  vrrp_gna_interval 0
}
vrrp_script chk_haproxy {
  script "killall -0 haproxy"
  interval 2
  weight 2
}
vrrp_instance haproxy-vip {
  state BACKUP
  priority 100
  interface eth0                       # Network card
  virtual_router_id 60
  advert_int 1
  authentication {
    auth_type PASS
    auth_pass 1111
  }
  unicast_src_ip 172.16.0.2      # The IP address of this machine
  unicast_peer {
    172.16.0.3                         # The IP address of peer machines
  }
  virtual_ipaddress {
    172.16.0.10/24                  # The VIP address
  }
  track_script {
    chk_haproxy
  }
}
备注

对于 interface 字段，您必须提供自己的网卡信息。您可以在机器上运行 ifconfig 以获取该值。

为 unicast_src_ip 提供的 IP 地址是您当前机器的 IP 地址。对于也安装了 HAproxy 和 Keepalived 进行负载均衡的其他机器，必须在字段 unicast_peer 中输入其 IP 地址。

保存文件并运行以下命令以重启 Keepalived。

systemctl restart keepalived
使 Keepalived 在开机后自动运行：

systemctl enable haproxy
确保您在另一台机器 (lb2) 上也配置了 Keepalived。

验证高可用
在开始创建 Kubernetes 集群之前，请确保已经测试了高可用。

在机器 lb1 上，运行以下命令：

[root@lb1 ~]# ip a s
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 52:54:9e:27:38:c8 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.2/24 brd 172.16.0.255 scope global noprefixroute dynamic eth0
       valid_lft 73334sec preferred_lft 73334sec
    inet 172.16.0.10/24 scope global secondary eth0 # The VIP address
       valid_lft forever preferred_lft forever
    inet6 fe80::510e:f96:98b2:af40/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
如上图所示，虚拟 IP 地址已经成功添加。模拟此节点上的故障：

systemctl stop haproxy
再次检查浮动 IP 地址，您可以看到该地址在 lb1 上消失了。

[root@lb1 ~]# ip a s
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 52:54:9e:27:38:c8 brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.2/24 brd 172.16.0.255 scope global noprefixroute dynamic eth0
       valid_lft 72802sec preferred_lft 72802sec
    inet6 fe80::510e:f96:98b2:af40/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
理论上讲，若配置成功，该虚拟 IP 会漂移到另一台机器 (lb2) 上。在 lb2 上运行以下命令，这是预期的输出：

[root@lb2 ~]# ip a s
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 52:54:9e:3f:51:ba brd ff:ff:ff:ff:ff:ff
    inet 172.16.0.3/24 brd 172.16.0.255 scope global noprefixroute dynamic eth0
       valid_lft 72690sec preferred_lft 72690sec
    inet 172.16.0.10/24 scope global secondary eth0   # The VIP address
       valid_lft forever preferred_lft forever
    inet6 fe80::f67c:bd4f:d6d5:1d9b/64 scope link noprefixroute
       valid_lft forever preferred_lft forever
如上所示，高可用已经配置成功。

### 存储部署 NFS服务器部署
搭建 NFS 服务器
Kubernetes 支持存储插件 NFS-client Provisioner。若想使用该插件，必须预先配置 NFS 服务器。NFS 服务器配置完成后，NFS 客户端会在服务器机器上挂载目录，以便 NFS 客户端访问 NFS 服务器上的文件，即您需要创建并输出客户端机器可以访问的目录。

NFS 服务器机器就绪后，您可以使用 KubeKey 通过 Helm Chart 来安装 NFS-client Provisioner 以及 Kubernetes 和 kcs。您必须在 Chart 配置中提供 NFS 服务器的输出目录以便 KubeKey 在安装时使用。

备注

您也可以在安装 Kubernetes 集群后创建 NFS-client 的存储类型。

本教程演示了如何安装 NFS 服务器，以 Ubuntu 16.04 为例。

安装及配置 NFS 服务器
步骤 1：安装 NFS 服务器 (NFS kernel server)
若要设置服务器机器，就必须在机器上安装 NFS 服务器。

运行以下命令，使用 Ubuntu 上的最新软件包进行安装。

sudo apt-get update
安装 NFS 服务器。

sudo apt install nfs-kernel-server
步骤 2：创建输出目录
NFS 客户端将在服务器机器上挂载一个目录，该目录已由 NFS 服务器输出。

运行以下命令来指定挂载文件夹名称（例如，/mnt/demo）。

sudo mkdir -p /mnt/demo
出于演示目的，请移除该文件夹的限制性权限，这样所有客户端都可以访问该目录。

sudo chown nobody:nogroup /mnt/demo
sudo chmod 777/mnt/demo
步骤 3：授予客户端机器访问 NFS 服务器的权限
运行以下命令：

sudo nano /etc/exports
将客户端信息添加到文件中。

/mnt/demo clientIP(rw,sync,no_subtree_check)
如果您有多台客户端机器，则可以将它们的客户端信息全部添加到文件中。或者，在文件中指定一个子网，以便该子网中的所有客户端都可以访问 NFS 服务器。例如：

/mnt/demo 192.168.0.0/24(rw,sync,no_subtree_check)
备注

rw：读写操作。客户端机器拥有对存储卷的读写权限。
sync：更改将被写入磁盘和内存中。
no_subtree_check：防止子树检查，即禁用客户端挂载允许的子目录所需的安全验证。
编辑完成后，请保存文件。

步骤 4：应用配置
运行以下命令输出共享目录。

sudo exportfs -a
重启 NFS 服务器。

sudo systemctl restart nfs-kernel-server

### addons 【持久化存储配置]
请在所有客户端上安装 nfs-common，它提供必要的 NFS 功能，而无需安装其他服务器组件。

执行以下命令确保使用最新软件包。

sudo apt-get update
在所有客户端上安装 nfs-common。

sudo apt-get install nfs-common

vi nfs-client.yaml
示例配置文件：

nfs:
  server: "192.168.0.2"    # This is the server IP address. Replace it with your own.
  path: "/mnt/demo"    # Replace the exported directory with your own.
storageClass:
  defaultClass: true
备注

storageClass.defaultClass 字段决定是否将 NFS-client Provisioner 的存储类型设置为默认存储类型。
保存文件。