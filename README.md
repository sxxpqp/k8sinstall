# *k8s 部署使用教程*
## 前提条件
* 安装并启动docker/containerd(1.20+安装containerd),已经安装了会重启docker/containerd. 高版本离线包自带docker/containerd，如没安装docker/containerd会自动安装.
* 下载kubernetes 离线安装包.
* 下载最新版本sealos.
* 务必同步服务器时间
   例：
```
yum install ntpdate -y
ntpdate time.windows.com
```
* 主机名不可重复 所有节点主机名
```
cat /etc/hostnames
```
  
* master节点CPU必须2C以上
* 请使用sealos 3.2.0以上版本
## 安装教程
1. 步骤一 安装kebernetes
* 多master 高可用
```
sealos init --master 192.168.0.2 \
    --master 192.168.0.3 \
    --master 192.168.0.4 \
    --node 192.168.0.5 \
    --user root \
    --passwd your-server-password \
    --version v1.14.1 \
    --pkg-url /root/kube1.14.1.tar.gz 
 ```   
* 单master多node
```
sealos init --master 192.168.0.2 \
    --node 192.168.0.5 \
    --user root \
    --passwd your-server-password \
    --version v1.14.1 \
    --pkg-url /root/kube1.14.1.tar.gz 
```
* 自定义ssh端口号,如2022：
```
sealos init --master 172.16.198.83:2022 \
    --pkg-url https://YOUR_HTTP_SERVER/kube1.15.0.tar.gz \
    --pk /root/kubernetes.pem \
    --version v1.15.0
```
2. 步骤二