```
k8s-master 192.168.80.81
k8s-node1 192.168.80.82
k8s-node2 192.168.80.83
```

###基础环境（每台机器上都需要执行）：

```
#关闭防火墙：
systemctl stop firewalld
systemctl disable firewalld
```

```
#关闭selinux：
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0                                     
```

```
#关闭swap：
swapoff -a  
sed -ir 's/.*swap.*/#&/' /etc/fstab   
cat /etc/fstab 
```

```
#设置主机名：
hostnamectl set-hostname -<hostname>-
```

```
#将桥接的IPv4流量传递到iptables的链：
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
#开启IP转发功能
echo "1" > /proc/sys/net/ipv4/ip_forward
sysctl --system 
```

```
#时间同步：
yum install ntpdate -y
ntpdate time.windows.com
```

```
#加载内核模块ipvs
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
modprobe -- br_netfilter
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules

```

```
#设置内核参数
cat << EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
sysctl -p /etc/sysctl.d/k8s.conf

```

### 安装docker（每台机器上都需要执行）：

```
#卸载旧版本docker
yum remove docker \
           docker-client \
           docker-client-latest \
           docker-common \
           docker-latest \
           docker-latest-logrotate \
           docker-logrotate \
           docker-selinux \
           docker-engine-selinux \
           docker-engine
#安装依赖包
yum install -y yum-utils device-mapper-persistent-data lvm2
#设置安装源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#开始安装
yum makecache fast
yum list docker-ce --showduplicates | sort -r
yum -y install docker-ce-20.10.3
systemctl start docker
systemctl enable docker
```

```
#另外Kubeadm建议将 systemd 设置为 cgroup 驱动，所以还要修改 daemon.json
sed -i "13i ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT"  /usr/lib/systemd/system/docker.service

tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://bk6kzfqm.mirror.aliyuncs.com"],
  "data-root": "/data/docker",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF

systemctl daemon-reload
systemctl restart docker
```
###安装kubeadm、kubelet

```
#配置安装源
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
#重建yum缓存，输入y添加证书认证
yum makecache fast
#安装
#yum install -y kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0（指定版本安装）
yum install -y kubelet kubeadm kubectl
systemctl enable --now kubelet
#安装bash自动补全插件
yum install bash-completion -y
#设置kubectl与kubeadm命令补全，下次login生效
kubectl completion bash > /etc/bash_completion.d/kubectl
kubeadm completion bash > /etc/bash_completion.d/kubeadm

```

```
#master初始化集群
kubeadm init \
--apiserver-advertise-address=172.16.5.10,1.2.3.4 \
--image-repository registry.aliyuncs.com/google_containers \
--kubernetes-version v1.18.0 \
--service-cidr=10.96.0.0/12 \
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=all

#kubeconfig文件
#把密钥配置加载到自己的环境变量里
export KUBECONFIG=/etc/kubernetes/admin.conf
#每次启动自动加载$HOME/.kube/config下的密钥配置文件（K8S自动行为）
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#集群网络安装
 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

#node加入集群
kubeadm join 192.168.80.80:6443 --token j96ea3.4e0z1sd76dpamb4sn2 \
    --discovery-token-ca-cert-hash sha256:b0ae45fb23da0oos0762e33001e60e7e6459522a33e8f6de46ef100ff0c65dcea62
```
