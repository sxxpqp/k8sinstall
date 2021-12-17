# k8s 部署使用教程
## 前提条件
* 部署linux系统 如：centos7 ubuntu18、20(LTS) suse等
* 安装并启动docker/containerd(1.20+安装containerd),已经安装了会重启docker/containerd. 高版本离线包自带docker/containerd，如没安装docker/containerd会自动安装.
* 默认安装是calio网络插件
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
    
    查看node节点状态是否ready
    ```
    kubectl get node
    ```
    ```
    NAME                      STATUS   ROLES    AGE     VERSION
    izj6cdqfqw4o4o9tc0q44rz   Ready    master   2m25s   v1.14.1
    zj6cdqfqw4o4o9tc0q44sz   Ready    master   119s    v1.14.1
    izj6cdqfqw4o4o9tc0q44tz   Ready    master   63s     v1.14.1
    izj6cdqfqw4o4o9tc0q44uz   Ready    <none>   38s     v1.14.1

    ``` 
    查看所有pod是否running    
    ```
    kebetctl get pod --all 

## 对集群删除、节点添加、节点删除
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
  
    **注意clean不加任何参数会清理整个**
```
    sealos clean --master 192.168.0.6 --master 192.168.0.7
    sealos clean --master 192.168.0.6-192.168.0.9  # 或者多个连续IP
```    

* 删除指定node节点
    ```
    sealos clean --node 192.168.0.6 --node 192.168.0.7
    sealos clean --node 192.168.0.6-192.168.0.9  # 或者多个连续IP
3. 步骤三 安装存储插件 （csi-nfs组件安装步骤）
   
   3.1 centos7配置nfs服务器 
    ```

    [root@localhost ~]#yum install nfs-utils rpcbind -y、
    [root@localhost ~]#mkdir -p /data/nfs/
    [root@localhost ~]#chmod 755 /data/nfs/
    [root@localhost ~]#vim /etc/exports
    /data/nfs/    *(insecure,rw,sync,no_root_squash,no_all_squash)    //nfs挂载目录；*：挂载地址（*代表所有）
    [root@localhost ~]#systemctl start nfs
    [root@localhost ~]#systemctl start rpcbind
    ```
   3.2 csi-nfs-driver插件
    ```
    cd csi-nfs
    bash install-dirver.sh
    ```

    3.3 修改storageclass-nfs.yaml
    ```
        ---
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
    name: nfs-csi
    provisioner: nfs.csi.k8s.io
    parameters:
    server: nfs-server.default.svc.cluster.local
    share: /
    reclaimPolicy: Delete
    volumeBindingMode: Immediate
    mountOptions:
    - hard
    - nfsvers=4.1
    ```
    **需要修改server、share路径**
    如：
    server：nfs-server
    share：/demo/data

    3.4 部署storageclass-nfs.yaml
    ```
    kubectl apply -f storageclass-nfs.yaml
    ```
    3.5 验证pod使用storageclass动态创建pvc.
    ```
    
    ````
    3.6 查看nfs服务器是否挂在
    ```
    cat /demo/data
    ```


4. 步骤四 安装前端组件
4.1执行以下命令开始安装：
    ```
    kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.2.0/kubesphere-installer.yaml
    ```
    ```
    kubectl apply -f https://github.com/kubesphere/ks-installer/releases/download/v3.2.0/cluster-configuration.yaml
    ```

    4.2 检查安装日志：
    ```
    kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f
    ```
    4.3 使用 
    ```
    kubectl get pod --all-namespaces 
    ```
    查看所有 Pod 是否在 KubeSphere 的相关命名空间中正常运行。如果是，请通过以下命令检查控制台的端口（默认为 30880）：
    ```
    kubectl get svc/ks-console -n kubesphere-system
    ```
    4.4 确保在安全组中打开了端口 30880，并通过 NodePort (IP:30880) 使用默认帐户和密码 (admin/P@88w0rd) 访问 Web 控制台。
 

    4.5 登录控制台后，您可以在系统组件中检查不同组件的状态。如果要使用相关服务，可能需要等待某些组件启动并运行。
5. 步骤五：安装显卡的插件（可选）前提是宿主机把显卡驱动安装好。
   5.1  部署NVIDIA docker2
    ```
    yum install nvidia-docker2
    sudo pkill -SIGHUP dockerd
    ```
   
   
   5.2 部署nvidia组件
    ```
    cd nvidia-device-plugin
    helm install nvidiaplugin .
    ```
    5.3 检查nvidia组件的状态
    ```
    kubectlg get pod -w -n kube-system
    ```
    5.4 pod验证nvidia是否正常
    ```
    cd ..
    kubectl apply -f pod-nvidia.yaml
    ```
    running为正常



