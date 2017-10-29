
#### 准备阶段
```
# 主机信息
k8s-master:192.168.1.30
k8s-node1:192.168.1.31

# 安装docker
# node1:
yum install docker
systemctl enable docker && systemctl start docker
```
## 安装master节点
#### 1、在master上安装kubeadm
```
# master节点安装
# centos 7
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
yum install -y docker kubelet kubeadm kubectl kubernetes-cni
systemctl enable docker && systemctl start docker
systemctl enable kubelet && systemctl start kubelet


# ubuntu 16.04
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF > /etc/apt/sources.list.d/kubernetes.list
deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update
# # Install docker if you don't have it already.
apt-get install -y docker.io
apt-get install -y kubelet kubeadm kubectl kubernetes-cni

# 注意：Centos 和 Ubuntu 的master 不一样，Centos的 master下需要安装docker，Ubuntu的master 不需要安装docker
```

#### 2、初始化kubernetes 
```
# 在Kubeadm 的文档中，Pod Network的安装是作为一个单独的步骤的。kubeadm init并没有选择一个默认的Pod network进行安装。这里采用Flannel 作为Pod network，如果我们要使用Flannel，那么在执行init时，按照kubeadm文档要求，我们必须给init命令带上option：–pod-network-cidr=10.244.0.0/16。如果有多网卡的，可以根据实际情况配置–api-advertise-addresses=，单网卡情况可以省略。多网卡的并没有验证过。

[root@k8s-master ~]# kubeadm init --kubernetes-version "v1.7.9" --pod-network-cidr=10.244.0.0/16     # 如果不指定版本，默认为最新版本
...
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run (as a regular user):

  sudo cp /etc/kubernetes/admin.conf $HOME/
  sudo chown $(id -u):$(id -g) $HOME/admin.conf
  export KUBECONFIG=$HOME/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  http://kubernetes.io/docs/admin/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join --token 8c7976.f2598fd46d21794f 192.168.1.30:6443
```



#### 3、验证初始化
```
  
sudo cp /etc/kubernetes/admin.conf $HOME/
sudo chown $(id -u):$(id -g) $HOME/admin.conf
export KUBECONFIG=$HOME/admin.conf
 
# 如果不执行上述命令则会出现以下报错
The connection to the server localhost:8080 was refused - did you specify the right host or port?

```

#### 4、检查安装
```
[root@k8s-master ~]# kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                 READY     STATUS    RESTARTS   AGE       IP             NODE
kube-system   etcd-k8s-master                      1/1       Running   0          2m        192.168.1.30   k8s-master
kube-system   kube-apiserver-k8s-master            1/1       Running   0          2m        192.168.1.30   k8s-master
kube-system   kube-controller-manager-k8s-master   1/1       Running   0          1m        192.168.1.30   k8s-master
kube-system   kube-dns-3913472980-p5ppd            0/3       Pending   0          2m        <none>         
kube-system   kube-proxy-724nd                     1/1       Running   0          2m        192.168.1.30   k8s-master
kube-system   kube-scheduler-k8s-master            1/1       Running   0          1m        192.168.1.30   k8s-master

# 注意：由于没有网络，所以出现kube-dns-3913472980-p5ppd            0/3       Pending
```


#### 5、安装flannel pod网络
```
[root@k8s-master ~]# kubectl create -f https://github.com/coreos/flannel/raw/master/Documentation/kube-flannel-rbac.yml
clusterrole "flannel" configured
clusterrolebinding "flannel" configured

[root@k8s-master ~]# kubectl create -f  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
serviceaccount "flannel" created
configmap "kube-flannel-cfg" created
daemonset "kube-flannel-ds" created

# 安装flannel需要下载flannel镜像，安装过程需要一定的时间。
```

#### 6、查看flannel的安装
```
[root@k8s-master ~]# kubectl get pod --all-namespaces -o wide
NAMESPACE     NAME                                 READY     STATUS    RESTARTS   AGE       IP             NODE
kube-system   etcd-k8s-master                      1/1       Running   0          10m       192.168.1.30   k8s-master
kube-system   kube-apiserver-k8s-master            1/1       Running   0          10m       192.168.1.30   k8s-master
kube-system   kube-controller-manager-k8s-master   1/1       Running   0          10m       192.168.1.30   k8s-master
kube-system   kube-dns-3913472980-p5ppd            3/3       Running   0          15m       10.244.0.2     k8s-master
kube-system   kube-flannel-ds-cl8vd                2/2       Running   0          10m       192.168.1.30   k8s-master
kube-system   kube-proxy-724nd                     1/1       Running   0          15m       192.168.1.30   k8s-master
kube-system   kube-scheduler-k8s-master            1/1       Running   0          10m       192.168.1.30   k8s-master

[root@k8s-master ~]# ps -ef | grep flannel
root       6952   6928  0 Jun05 ?        00:00:01 /opt/bin/flanneld --ip-masq --kube-subnet-mgr
root      10263  10251  0 Jun05 ?        00:00:00 /bin/sh -c set -e -x; cp -f /etc/kube-flannel/cni-conf.json /etc/cni/net.d/10-flannel.conf; while true; do sleep 3600; done
root      14935   8344  0 00:21 pts/0    00:00:00 grep --color=auto flannel

```

## 加入node节点
#### 1、安装必要的软件
```
# 同 master 1
# 提示：node节点也需要安装 docker kubelet kubeadm kubectl kubernetes-cni
```

#### 2、查看master token，然后加入node
```
[root@k8s-master ~]# kubeadm token list
TOKEN                     TTL         EXPIRES   USAGES                   DESCRIPTION
8c7976.f2598fd46d21794f   <forever>   <never>   authentication,signing   The default bootstrap token generated by 'kubeadm init'.

[root@k8s-node1 ~]# kubeadm join --token=8c7976.f2598fd46d21794f 192.168.1.30:6443
Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```

#### 3、验证集群状态
```
[root@k8s-master ~]# kubectl get nodes;
NAME         STATUS    AGE       VERSION
k8s-master   Ready     42m       v1.6.4
k8s-node1    NotReady   35s       v1.6.4

# NotReady? What's Fucking? 按照官网的应该Ready
```

##### 3.1 排错查看是否容器出现问题
```
[root@k8s-master ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY     STATUS              RESTARTS   AGE
kube-system   etcd-k8s-master                      1/1       Running             1          18h
kube-system   kube-apiserver-k8s-master            1/1       Running             1          18h
kube-system   kube-controller-manager-k8s-master   1/1       Running             1          18h
kube-system   kube-dns-3913472980-p5ppd            3/3       Running             10         18h
kube-system   kube-flannel-ds-cl8vd                2/2       Running             9          18h
kube-system   kube-flannel-ds-m9lvh                0/2       ContainerCreating   0          3m
kube-system   kube-proxy-724nd                     1/1       Running             1          18h
kube-system   kube-proxy-p1jls                     0/1       ContainerCreating   0          3m
kube-system   kube-scheduler-k8s-master            1/1       Running             1          18h

# 果然不出所料
```

##### 3.2 查看具体原因
```
[root@k8s-master ~]# kubectl describe pod kube-flannel-ds-m9lvh --namespace=kube-system

  # 仔细一看，原来是不能翻墙，导致镜像拉取不到，日了狗了
```

##### 3.3 拉取镜像pause-amd64:3.0到本地，然后打上tag
```
[root@k8s-node1 ~]# docker pull sugare/pause-amd64:3.0
[root@k8s-node1 ~]# docker tag sugare/pause-amd64:3.0 gcr.io/google_containers/pause-amd64:3.0
[root@k8s-node1 ~]# docker pull sugare/flannel:v0.7.1-amd64
[root@k8s-node1 ~]# docker tag sugare/flannel:v0.7.1-amd64 quay.io/coreos/flannel:v0.7.1-amd64
[root@k8s-node1 ~]# docker pull sugare/kube-proxy-amd64:v1.6.4
[root@k8s-node1 ~]# docker tag sugare/kube-proxy-amd64 gcr.io/google_containers/kube-proxy-amd64:v1.6.4

```

##### 3.4 查看node状态
```
[root@k8s-master ~]# kubectl get nodes
NAME         STATUS    AGE       VERSION
k8s-master   Ready     19h       v1.6.4
k8s-node1    Ready     17h       v1.6.4

[root@k8s-master ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY     STATUS              RESTARTS   AGE
kube-system   etcd-k8s-master                      1/1       Running             1          18h
kube-system   kube-apiserver-k8s-master            1/1       Running             1          18h
kube-system   kube-controller-manager-k8s-master   1/1       Running             1          18h
kube-system   kube-dns-3913472980-p5ppd            3/3       Running             10         18h
kube-system   kube-flannel-ds-cl8vd                2/2       Running             9          18h
kube-system   kube-flannel-ds-m9lvh                0/2       Running   0          3m
kube-system   kube-proxy-724nd                     1/1       Running             1          18h
kube-system   kube-proxy-p1jls                     0/1       Running   0          3m
kube-system   kube-scheduler-k8s-master            1/1       Running             1          18h

# 注意：如果node的 selinux 未关出现 CrashLoopBackOff  情况
```

#### 将maste节点设为可调度的节点（可选）
```
# kubectl taint nodes --all node-role.kubernetes.io/master-
```

#### 不在master节点上操作集群，而是在其他工作节点上操作集群（可选）
```
# scp root@<master ip>:/etc/kubernetes/admin.conf .  

# kubectl --kubeconfig ./admin.conf get nodes
```

#### 参考文档
https://kubernetes.io/docs/setup/independent/install-kubeadm/

https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/

kubeadm 1.5:
http://www.dnsdizhi.com/post-278.html

kubeadm 1.6: http://blog.csdn.net/ximenghappy/article/details/70157361








