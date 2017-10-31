### 0. 相关镜像

镜像名 | 版本号
---|---
quay.io/coreos/etcd | v3.2.4
quay.io/calico/ctl | v1.5.0
quay.io/calico/node | v2.5.0
quay.io/calico/cni | v1.10.0
quay.io/calico/kube-policy-controller | v0.7.0
quay.io/calico/routereflector | v0.4.0
quay.io/coreos/hyperkube | v1.8.1_coreos.0
gcr.io/google_containers/pause-amd64 | 3.0
gcr.io/google_containers/k8s-dns-kube-dns-amd64 | 1.14.5
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64 | 1.14.5
gcr.io/google_containers/k8s-dns-sidecar-amd64 | 1.14.5
sugare/cluster-proportional-autoscaler-amd64 | latest
lachlanevenson/k8s-helm | v2.2.2
gcr.io/kubernetes-helm/tiller | v2.2.2
gcr.io/google_containers/elasticsearch | v2.4.1
gcr.io/google_containers/fluentd-elasticsearch | 1.22
gcr.io/google_containers/kibana | v4.6.1

### 1. 下载 kubespray
```
# git clone https://github.com/kubernetes-incubator/kubespray.git
# cd kubespray/
```

### 2. 更改源
```
# vim changY.sh
#!/bin/bash
#
## kubespray 2.1
# grc_image_files=(
# ./extra_playbooks/roles/dnsmasq/templates/dnsmasq-autoscaler.yml
# ./extra_playbooks/roles/download/defaults/main.yml
# ./extra_playbooks/roles/kubernetes-apps/ansible/defaults/main.yml
# ./roles/download/defaults/main.yml
# ./roles/dnsmasq/templates/dnsmasq-autoscaler.yml
# ./roles/kubernetes-apps/ansible/defaults/main.yml
# )

## kubespray 2.3
grc_image_files=(
./extra_playbooks/roles/dnsmasq/templates/dnsmasq-autoscaler.yml.j2
./extra_playbooks/roles/download/defaults/main.yml
./extra_playbooks/roles/kubernetes-apps/ansible/defaults/main.yml
./roles/download/defaults/main.yml
./roles/dnsmasq/templates/dnsmasq-autoscaler.yml.j2 
./roles/kubernetes-apps/ansible/defaults/main.yml
)

for file in ${grc_image_files[@]} ; do
	# echo $file $file.bak
	/bin/cp $file $file.bak
	sed -i 's#gcr.io/google_containers#sugare#g' $file
	sed -i 's#gcr.io/kubernetes-helm#sugare#g' $file
	sed -i 's#quay.io/calico#sugare#g' $file
done

# sh changY.sh

```

### 2. 根据情况加入节点，创建配置文件
```
# CONFIG_FILE=inventory/inventory.cfg python3 contrib/inventory_builder/inventory.py 192.168.0.5 192.168.0.6 

```

### 3. 开始安装
```
# ansible-playbook -i inventory/inventory.cfg cluster.yml -b -v --private-key=~/.ssh/id_rsa
```

### 4. 错误回滚
```
ansible-playbook -i inventory/inventory.cfg reset.yml -b -v --private-key=~/.ssh/id_rsa
```



### 附录：
####  错误1. swap报错
```
报错提示：
TASK [kubernetes/preinstall : Stop if swap enabled] *********************************************************
Saturday 28 October 2017  15:48:41 +0800 (0:00:00.074)       0:00:14.596 ****** 
fatal: [node1]: FAILED! => {
    "assertion": "ansible_swaptotal_mb == 0", 
    "changed": false, 
    "evaluated_to": false, 
    "failed": true
}
```

解决方法：
```
# vim ./roles/kubernetes/preinstall/defaults/main.yml 
...

# Set to true to allow pre-checks to fail and continue deployment
ignore_assert_errors: true  # 改为 true
```

#### 错误2 swap on is not supported
```
错误提示：
Kubelet: Running with swap on is not supported, please disable swap! or set --fail-swap-on flag to false. /proc/

# 提示：Kubernetes 1.8开始要求关闭系统的Swap，如果不关闭，默认配置下kubelet将无法启动。可以通过kubelet的启动参数–fail-swap-on=false更改这个限制。 我们这里关闭系统的Swap:
```

解决方法：
```
swapoff -a

注意：安装结束后
swapon -a

```

#### 错误3 解决 kubedns-autoscaler无法启动
```
# cd /etc/kubernetes/
# sed -i 's#sugare/cluster-proportional-autoscaler-amd64:1.1.1#sugare/cluster-proportional-autoscaler-amd64#g' kubedns-autoscaler.yml.j2
```

#### 错误4 出现 dial tcp 10.233.102.191:8081: getsockopt: connection refused
```
错误提示：  Normal   Started                2m               kubelet, node1     Started container
  Normal   Created                2m               kubelet, node1     Created container
  Normal   Created                2m               kubelet, node1     Created container
  Normal   Started                2m               kubelet, node1     Started container
  Warning  Unhealthy              1m               kubelet, node1     Liveness probe failed: HTTP probe failed with statuscode: 503
  Normal   Created                1m (x2 over 2m)  kubelet, node1     Created container
  Normal   Pulled                 1m (x2 over 2m)  kubelet, node1     Container image "sugare/k8s-dns-kube-dns-amd64:1.14.5" already present on machine
  Normal   Started                1m (x2 over 2m)  kubelet, node1     Started container
  Warning  Unhealthy              1m (x7 over 2m)  kubelet, node1     Readiness probe failed: Get http://10.233.102.191:8081/readiness: dial tcp 10.233.102.191:8081: getsockopt: connection refused
  Warning  Unhealthy              1m (x3 over 1m)  kubelet, node1     Liveness probe failed: HTTP probe failed with statuscode: 503
```
解决方法：
```
systemctl stop firewalld
systemctl disable firewalld
```
#### 失败清理（推荐使用错误回滚）
```
rm -rf /etc/kubernetes/
rm -rf /var/lib/kubelet
rm -rf /var/lib/etcd
rm -rf /usr/local/bin/kubectl
rm -rf /etc/systemd/system/calico-node.service
rm -rf /etc/systemd/system/kubelet.service
systemctl stop etcd.service
systemctl disable etcd.service
systemctl stop calico-node.service
systemctl disable calico-node.service
docker stop $(docker ps -q)
docker rm $(docker ps -a -q)
service docker restart
```

#### 参考：
http://blog.csdn.net/zhuchuangang/article/details/77712614



