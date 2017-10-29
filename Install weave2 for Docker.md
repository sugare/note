#### 安装 weave 命令行工具
```
curl -L git.io/weave -o /usr/local/bin/weave
chmod +x /usr/local/bin/weave
```

#### 启动 weave 虚拟路由器，每个weave网络内的主机上都要运行
```
[root@zoo1 ~]# weave launch
1db06cf41958373ab73c30c85f68e80b47ae7b4c7d2b3610c934dcf0b9034040
[root@zoo1 ~]# docker ps
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS               NAMES
1db06cf41958        weaveworks/weave:2.0.4   "/home/weave/weave..."   8 minutes ago       Up 7 minutes                            weave

```

#### 设置环境变量，这样通过docker命令启动的容器就会自动连接到weave网络中
```
[root@zoo1 ~]# eval $(weave env)
```

#### 启动容器查看是否使用weave网络
```
[root@zoo1 ~]# docker run --name w1 -ti busybox
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
45: eth0@if46: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link 
       valid_lft forever preferred_lft forever
47: ethwe@if48: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1376 qdisc noqueue 
    link/ether d6:3d:45:f7:b6:20 brd ff:ff:ff:ff:ff:ff
    inet 10.32.0.1/12 scope global ethwe    ## 成功引用weave网络
       valid_lft forever preferred_lft forever
    inet6 fe80::d43d:45ff:fef7:b620/64 scope link tentative flags 08 
       valid_lft forever preferred_lft forever
```


#### 在zoo2上创建远端连接并配置环境变量
```
[root@zoo2 ~]# weave launch zoo1
ab229c31d856c33d53894232e65bf4208fad50049e0df32b16f37c77b285ba56
[root@zoo2 ~]# docker ps
CONTAINER ID        IMAGE                    COMMAND                  CREATED              STATUS              PORTS               NAMES
ab229c31d856        weaveworks/weave:2.0.4   "/home/weave/weave..."   About a minute ago   Up 36 seconds                           weave
提示：
    指定多个远端主机用空格隔开：
    $ weave launch <ip address> <ip address> 

[root@zoo2 ~]# eval $(weave env)

```

#### 在zoo2启动容器
```
[root@zoo2 ~]# docker run --name w2 -ti busybox
/ # ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
42: eth0@if43: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link 
       valid_lft forever preferred_lft forever
44: ethwe@if45: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1376 qdisc noqueue 
    link/ether c6:15:cd:52:be:75 brd ff:ff:ff:ff:ff:ff
    inet 10.32.0.2/12 scope global ethwe        ## IP分配成功
       valid_lft forever preferred_lft forever
    inet6 fe80::c415:cdff:fe52:be75/64 scope link tentative flags 08 
       valid_lft forever preferred_lft forever

```

#### 检查是否可以互通 在w2 中 ping w1
```
/ # ping -c 1 w1
PING w1 (10.32.0.1): 56 data bytes
64 bytes from 10.32.0.1: seq=0 ttl=64 time=0.588 ms     ## 完美

--- w1 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 0.588/0.588/0.588 ms
```

#### 指定IP创建容器
```
[root@zoo2 ~]# docker run -e WEAVE_CIDR=10.32.0.100/12 --name custom_ip -it busybox
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
21: eth0@if22: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.2/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe11:2/64 scope link 
       valid_lft forever preferred_lft forever
23: ethwe@if24: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1376 qdisc noqueue 
    link/ether ba:1a:29:26:4f:3a brd ff:ff:ff:ff:ff:ff
    inet 10.32.0.100/12 scope global ethwe
       valid_lft forever preferred_lft forever
    inet6 fe80::b81a:29ff:fe26:4f3a/64 scope link tentative flags 08 
       valid_lft forever preferred_lft forever

```

#### 同一段访问
```
/ # hostname 
w1.weave.local
/ # 
/ # ping -c 1 10.32.0.100
PING 10.32.0.100 (10.32.0.100): 56 data bytes
64 bytes from 10.32.0.100: seq=0 ttl=64 time=2.461 ms

--- 10.32.0.100 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 2.461/2.461/2.461 ms

```

#### 外部访问
```
## 默认 ping 不通
[root@zoo2 ~]# ping -c 1 10.32.0.100
PING 10.32.0.100 (10.32.0.100) 56(84) bytes of data.
^C
--- 10.32.0.100 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms

## 通过 expose 将其暴露出来
[root@zoo2 ~]# weave expose
10.32.0.4   // 暴露出来会给weave 网桥随机分配一个IP

## 也可以通过如下方法暴露
[root@zoo2 ~]# weave expose net:default net:10.32.0.0/12

## 通过 命令查看
[root@zoo2 ~]# ifconfig weave
weave: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1376
        inet 10.32.0.4  netmask 255.240.0.0  broadcast 0.0.0.0
        inet6 fe80::6c86:f3ff:fef1:ddaa  prefixlen 64  scopeid 0x20<link>
        ether 6e:86:f3:f1:dd:aa  txqueuelen 1000  (Ethernet)
        RX packets 19  bytes 1096 (1.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 12  bytes 872 (872.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


## 再次 ping，网桥都建立了，肯定通
[root@zoo2 ~]# ping -c 1 10.32.0.100
PING 10.32.0.100 (10.32.0.100) 56(84) bytes of data.
64 bytes from 10.32.0.100: icmp_seq=1 ttl=64 time=0.189 ms

--- 10.32.0.100 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.189/0.189/0.189/0.000 ms
```
#### ping另一个主机的容器
```
[root@zoo2 ~]# ping $(weave dns-lookup w1)
PING 10.32.0.1 (10.32.0.1) 56(84) bytes of data.
64 bytes from 10.32.0.1: icmp_seq=1 ttl=64 time=0.802 ms
64 bytes from 10.32.0.1: icmp_seq=2 ttl=64 time=0.746 ms
64 bytes from 10.32.0.1: icmp_seq=3 ttl=64 time=0.850 ms
64 bytes from 10.32.0.1: icmp_seq=4 ttl=64 time=0.893 ms
^C
--- 10.32.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3001ms
rtt min/avg/max/mdev = 0.746/0.822/0.893/0.065 ms
```

#### 隐藏网络
```
[root@zoo2 ~]# weave hide
10.32.0.4

[root@zoo2 ~]# ifconfig weave       ## 无IP已隐藏
weave: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1376
        inet6 fe80::6c86:f3ff:fef1:ddaa  prefixlen 64  scopeid 0x20<link>
        ether 6e:86:f3:f1:dd:aa  txqueuelen 1000  (Ethernet)
        RX packets 23  bytes 1340 (1.3 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 12  bytes 872 (872.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


```


参考自：

http://lib.csdn.net/article/docker/67194

