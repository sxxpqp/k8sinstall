# k8s 部署使用教程
## 前提条件
* 部署linux系统 如：centos7 ubuntu18、20(LTS) suse等
* 安装并启动docker/containerd(1.20+安装containerd),已经安装了会重启docker/containerd. 高版本离线包自带docker/containerd，如没安装docker/containerd会自动安装.
* 下载kubernetes 离线安装包.
* 务必同步服务器时间
    ```
    yum install ntpdate -y
    ntpdate time.windows.com
* 主机名不可重复 所有节点主机名
    ```
    cat /etc/hostnames  
* master节点CPU必须2C以上
* 开启ssh远程连接账号及密码。如账号：root 密码：123456
* 请使用sealos 3.2.0以上版本
    ```
    wget -c https://sealyun.oss-cn-beijing.aliyuncs.com/latest/sealos &&chmod +x sealos && mv sealos /usr/bin 
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
   
* 单master多node
    ```
    sealos init --master 192.168.0.2 \
    --node 192.168.0.5 \
    --user root \
    --passwd your-server-password \
    --version v1.14.1 \
    --pkg-url /root/kube1.14.1.tar.gz

* 自定义ssh端口号,如2022：
    ```
    sealos init --master 172.16.198.83:2022 \
    --pkg-url https://YOUR_HTTP_SERVER/kube1.15.0.tar.gz \
    --pk /root/kubernetes.pem \
    --version v1.15.0



2. 步骤二  检查安装是否正常: 
    ```
    kubectl get node
    ```
    * NAME                      STATUS   ROLES    AGE     VERSION
    * izj6cdqfqw4o4o9tc0q44rz   Ready    master   2m25s   v1.14.1
    * izj6cdqfqw4o4o9tc0q44sz   Ready    master   119s    v1.14.1
    * izj6cdqfqw4o4o9tc0q44tz   Ready    master   63s     v1.14.1
    * izj6cdqfqw4o4o9tc0q44uz   Ready    <none>   38s     v1.14.1
    
    Ready 代表安装成功
        
    ```
    kebetctl get pod --all 
    查看所有pod是否running

## 对集群节点添加
* 删除kubernetes
    ```
    sealos clean --all


* 增加master
    ```
    sealos join --master 192.168.0.6 --master 192.168.0.7
    sealos join --master 192.168.0.6-192.168.0.9  # 或者多个连续IP


* 增加node
    ```
    sealos join --node 192.168.0.6 --node 192.168.0.7
    sealos join --node 192.168.0.6-192.168.0.9  # 或者多个连续IP

* 删除指定master节点
**注意clean不加任何参数会清理整个集群**
    ```
    sealos clean --master 192.168.0.6 --master 192.168.0.7
    sealos clean --master 192.168.0.6-192.168.0.9  # 或者多个连续IP


* 删除指定node节点
    ```
    sealos clean --node 192.168.0.6 --node 192.168.0.7
    sealos clean --node 192.168.0.6-192.168.0.9  # 或者多个连续IP
