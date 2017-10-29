#### 环境介绍

hostname | IP | role
---|---|---
zoo1 | 192.168.70.134 | calico01
zoo2 | 192.168.70.135 | calico02 
zoo3 | 192.168.70.3   | etcd


#### etcd 集群搭建
```
## etcd1
docker run -d -p 2379:2379 -p 2380:2380 --name calico_etcd elcolio/etcd \
-name etcd1 \
-advertise-client-urls http://192.168.70.3:2379 \
-listen-client-urls http://0.0.0.0:2379 \
-initial-advertise-peer-urls http://192.168.70.3:2380 \
-listen-peer-urls http://0.0.0.0:2380 \
-initial-cluster-token etcd-cluster \
-initial-cluster "etcd1=http://192.168.70.3:2380" \
-initial-cluster-state new

## 本例只使用一台 etcd （集群搭建略）
```

#### 查看 etc 集群状态
```
[root@zoo3 ~]# curl 192.168.70.3:2379/version
etcd 2.0.10

```

#### 配置 docker --cluster-store
```
## 目的是是使多台主机的 docker 使用同一 store
# egrep -v "^$|#" /usr/lib/systemd/system/docker.service 
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target firewalld.service
[Service]
Type=notify
ExecStart=/usr/bin/dockerd --cluster-store=etcd://192.168.70.3:2379
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target

## 提示：也可通过 dockerd 来设置 --cluster-store，如下：
dockerd \
    --cluster-store etcd://192.168.1.2:2379
    

```
 #### 安装 calico 命令行工具 calicoctl
 ```
 sudo wget -O /usr/local/bin/calicoctl https://github.com/projectcalico/calicoctl/releases/download/v1.5.0/calicoctl
sudo chmod +x /usr/local/bin/calicoctl

 ```
 
 #### 启动 calico-node
 ```
 ## 需要先指定存储
 ETCD_ENDPOINTS=http://192.168.70.3:2379 calicoctl node run --node-image=quay.io/calico/node:v2.5.1

提示：
1. 可修改文件 /etc/calico/calicoctl.cfg    默认calicoctl 先读取该文件
# cat /etc/calico/calicoctl.cfg 
apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: "http://192.168.70.3:2379"

然后通过 
calicoctl node run --node-image=quay.io/calico/node:v2.5.1 启动

2. 指定存储来启动，否则出现报错：
Skipping datastore connection test
ERROR: Unable to access datastore to query node configuration
Terminating
Error executing command: calico/node has terminated, check logs for details
 ```
 
 #### 检查BGP peers状态
 ```
 ## 正确状态
 [root@zoo1 ~]# calicoctl node status
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+----------+-------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+----------------+-------------------+-------+----------+-------------+
| 192.168.70.135 | node-to-node mesh | up    | 02:21:48 | Established |
+----------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

[root@zoo2 ~]# calicoctl node status
Calico process is running.

IPv4 BGP status
+----------------+-------------------+-------+----------+-------------+
|  PEER ADDRESS  |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+----------------+-------------------+-------+----------+-------------+
| 192.168.70.134 | node-to-node mesh | up    | 02:21:47 | Established |
+----------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

提示：如果INFO为 Established 状态则 calico 正常工作
## 如果出现 INFO: Active Socket:Connection closed 如：
[root@zoo1 docker]# calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+--------------------------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |              INFO              |
+--------------+-------------------+-------+----------+--------------------------------+
| 172.18.0.1   | node-to-node mesh | start | 14:30:36 | Active Socket: Connection      |
|              |                   |       |          | closed                         |
+--------------+-------------------+-------+----------+--------------------------------+

排查过程：
1. 检查防火墙
2. 检查 etcd
3. 检查路由情况

```

#### 基于calico driver创建三个不同网络
```
docker network create --driver calico --ipam-driver calico-ipam net1
docker network create --driver calico --ipam-driver calico-ipam net2
docker network create --driver calico --ipam-driver calico-ipam net3
```

#### 在 zoo1 创建三个容器
```
docker run --net net1 --name workload-A -tid busybox
docker run --net net2 --name workload-B -tid busybox
docker run --net net1 --name workload-C -tid busybox
```

#### 在 zoo2 创建三个容器
```
docker run --net net3 --name workload-D -tid busybox
docker run --net net1 --name workload-E -tid busybox
```

#### 路由情况
```
当在 zoo2 创建与 zoo1 处于同一docker network 时会自动创建路由来连接 zoo1 和 zoo2
[root@zoo1 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.70.2    0.0.0.0         UG    100    0        0 ens33
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.37.128  192.168.70.135  255.255.255.192 UG    0      0        0 ens33
192.168.70.0    0.0.0.0         255.255.255.0   U     100    0        0 ens33
192.168.179.192 0.0.0.0         255.255.255.255 UH    0      0        0 calie7e36ab2153
192.168.179.192 0.0.0.0         255.255.255.192 U     0      0        0 *
192.168.179.193 0.0.0.0         255.255.255.255 UH    0      0        0 calia84d11d74ab
192.168.179.194 0.0.0.0         255.255.255.255 UH    0      0        0 calif5ef14b77bb

[root@zoo2 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.70.2    0.0.0.0         UG    100    0        0 ens33
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
192.168.37.128  0.0.0.0         255.255.255.255 UH    0      0        0 cali19625a99bf1
192.168.37.128  0.0.0.0         255.255.255.192 U     0      0        0 *
192.168.37.129  0.0.0.0         255.255.255.255 UH    0      0        0 cali1622aee4f2f
192.168.70.0    0.0.0.0         255.255.255.0   U     100    0        0 ens33
192.168.179.192 192.168.70.134  255.255.255.192 UG    0      0        0 ens33

路由出现则可以通信
```


#### 通信情况
```
## 同一主机同一 docker network 容器通信情况
[root@zoo1 ~]# docker exec workload-A ping -c 4 workload-C.net1
PING workload-C.net1 (192.168.179.194): 56 data bytes
64 bytes from 192.168.179.194: seq=0 ttl=63 time=0.099 ms
64 bytes from 192.168.179.194: seq=1 ttl=63 time=0.169 ms
64 bytes from 192.168.179.194: seq=2 ttl=63 time=0.213 ms
64 bytes from 192.168.179.194: seq=3 ttl=63 time=0.102 ms

--- workload-C.net1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.099/0.145/0.213 ms

## 同一主机不同 docker network 容器通信情况
[root@zoo1 ~]# docker exec workload-A ping -c 2  `docker inspect --format "{{ .NetworkSettings.Networks.net2.IPAddress }}" workload-B`
PING 192.168.179.193 (192.168.179.193): 56 data bytes

--- 192.168.179.193 ping statistics ---
2 packets transmitted, 0 packets received, 100% packet loss

## 不同主机同一 docker network 容器通信情况
[root@zoo1 ~]# docker exec workload-A ping -c 4 workload-E.net1
PING workload-E.net1 (192.168.37.129): 56 data bytes
64 bytes from 192.168.37.129: seq=0 ttl=62 time=0.988 ms
64 bytes from 192.168.37.129: seq=1 ttl=62 time=0.605 ms
64 bytes from 192.168.37.129: seq=2 ttl=62 time=0.495 ms
64 bytes from 192.168.37.129: seq=3 ttl=62 time=0.510 ms

--- workload-E.net1 ping statistics ---
4 packets transmitted, 4 packets received, 0% packet loss
round-trip min/avg/max = 0.495/0.649/0.988 ms

```

#### 指定IP段创建ipPool
```
# cat << EOF | calicoctl create -f -
- apiVersion: v1
  kind: ipPool
  metadata:
    cidr: 192.0.2.0/24
EOF
```
#### 根据指定ipPool创建  docker network
```
[root@zoo1 ~]# docker network create --driver calico --ipam-driver calico-ipam --subnet=192.0.2.0/24 my_net
017ba1ce288983993db9ba28eec32052fb027d2328b521f885ce7f8ea7afcf3b

[root@zoo1 ~]# calicoctl get pool
CIDR                       
192.0.2.0/24               // 新建的 CIDR
192.168.0.0/16             
fd80:24e2:f998:72d6::/64 

[root@zoo1 ~]# docker network inspect my_net
[
    {
        "Name": "my_net",
        "Id": "017ba1ce288983993db9ba28eec32052fb027d2328b521f885ce7f8ea7afcf3b",
        "Created": "2017-09-20T10:57:32.669771969+08:00",
        "Scope": "global",
        "Driver": "calico",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "calico-ipam",
            "Options": {},
            "Config": [
                {
                    "Subnet": "192.0.2.0/24"        // 检查subnet
                }
            ]
        },
    }
]
```

#### 创建容器指定IP
```
[root@zoo1 ~]# docker run --net my_net --name zoo1_workload --ip 192.0.2.100 -tid busybox
0cea35841b74074098a158e6ff0b87736b95a8710a577e70dfb92d98f1754151

[root@zoo1 ~]# docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' zoo1_workload
192.0.2.100
```

#### 附录:
http://blog.csdn.net/yunqishequ1/article/details/54345380

https://docs.projectcalico.org/v2.5/introduction/

https://docs.projectcalico.org/v2.5/getting-started/docker/tutorials/security-using-calico-profiles

https://docs.projectcalico.org/master/getting-started/docker/installation/manual

