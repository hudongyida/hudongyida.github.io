---
layout    : post
title     : "Docker Bridge 网络 (2025)"
date      : 2025-04-16
lastupdate: 2025-04-16
categories: linux
---



## Docker bridge

下面将基于 **Linux 底层网络知识**，逐步演示如何**“手搓”（手动搭建）**一个 **Docker 容器 Bridge 网络**。

```bash
# 创建三个网络命名空间：docker、nginx 和 gitlab。
# 其中，docker 命名空间模拟主网络环境，作为基础网络栈；而 
# nginx 和 gitlab 命名空间则模拟两个独立容器的网络环境，实现隔离。
ip netns add docker
ip netns add nginx
ip netns add gitlab

# 为 docker、nginx 和 gitlab 三个网络命名空间分别启动 lo 接口
ip netns exec docker ip link set lo up
ip netns exec nginx  ip link set lo up
ip netns exec gitlab ip link set lo up

# 首先 添加一张物理或虚拟网卡，确保该网卡能够访问外部网络。
# 接着，将这张网卡从 主网络命名空间 移动到 docker 网络命名空间，以便为容器环境提供外部网络连接能力。
ip link set ens41 netns docker

# 进入 docker 网络命名空间，为 ens41 网卡配置 IP 地址并设置路由。
ip netns exec docker bash
ip address add 192.168.54.230/24 dev ens41
ip link set ens41 up
ip route add default via 192.168.54.51 dev ens41

# 创建并配置桥接设备 docker0，设置 IP 地址并激活设备。
ip link add docker0 type bridge
ip address add 172.17.0.1/16 dev docker0
ip link set docker0 up

# 创建一个 veth 对，将其中一端接入 docker0 桥接设备，另一端移动到 nginx 的网络命名空间。
ip link add veth-nginx type veth peer name eth0
ip link set veth-nginx master docker0
ip link set veth-nginx up
ip link set eth0 netns nginx

# 创建一个 veth 对，将其中一端接入 docker0 桥接设备，另一端移动到 gitlab 的网络命名空间。
ip link add veth-gitlab type veth peer name eth0
ip link set veth-gitlab master docker0
ip link set veth-gitlab up
ip link set eth0 netns gitlab

# 进入 nginx 的网络命名空间，为 eth0 接口 配置 IP 地址 并设置 默认路由。
ip netns exec nginx bash
ip address add 172.17.10.10/16 dev eth0
ip link set eth0 up
ip route add default via 172.17.0.1 dev eth0

# 进入 gitlab 的网络命名空间，为 eth0 接口 配置 IP 地址 并设置 默认路由。
ip netns exec gitlab bash
ip address add 172.17.10.20/16 dev eth0
ip link set eth0 up
ip route add default via 172.17.0.1 dev eth0
```

<br>

查看 **docker** 网络命名空间中的接口状况

```bash
root@localhost:~# ip netns exec docker ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 16:04:30:73:11:eb brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::d815:ffff:feef:cdb5/64 scope link 
       valid_lft forever preferred_lft forever
4: veth-nginx@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default qlen 1000
    link/ether 1e:d6:88:3d:97:93 brd ff:ff:ff:ff:ff:ff link-netns nginx
    inet6 fe80::1cd6:88ff:fe3d:9793/64 scope link 
       valid_lft forever preferred_lft forever
6: veth-gitlab@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default qlen 1000
    link/ether 16:04:30:73:11:eb brd ff:ff:ff:ff:ff:ff link-netns gitlab
    inet6 fe80::1404:30ff:fe73:11eb/64 scope link 
       valid_lft forever preferred_lft forever
17: ens41: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:24:d9:ac brd ff:ff:ff:ff:ff:ff
    altname enp2s9
    inet 192.168.54.230/24 scope global ens41
       valid_lft forever preferred_lft forever
    inet6 2409:8950:eba:47:20c:29ff:fe24:d9ac/64 scope global dynamic mngtmpaddr 
       valid_lft 3306sec preferred_lft 3306sec
    inet6 fe80::20c:29ff:fe24:d9ac/64 scope link 
       valid_lft forever preferred_lft forever
```


<br>

查看 **nginx** 网络命名空间中的接口状况

```bash
root@localhost:~# ip netns exec nginx ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
3: eth0@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 32:0e:7c:ac:eb:09 brd ff:ff:ff:ff:ff:ff link-netns docker
    inet 172.17.10.10/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::300e:7cff:feac:eb09/64 scope link 
       valid_lft forever preferred_lft forever
```

<br>

查看 **gitlab** 网络命名空间中的接口状况

```bash
root@localhost:~# ip netns exec gitlab ip a 
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
5: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 9e:e4:5c:ed:77:4f brd ff:ff:ff:ff:ff:ff link-netns docker
    inet 172.17.10.20/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::9ce4:5cff:feed:774f/64 scope link 
       valid_lft forever preferred_lft forever
```

<br>

**验证网络连通性**：测试 **docker0 网桥** 与 **nginx 的 eth0**、**gitlab 的 eth0** 之间的通信是否正常。

```bash
root@localhost:/# ping 172.17.10.10 -c 2
PING 172.17.10.10 (172.17.10.10) 56(84) bytes of data.
64 bytes from 172.17.10.10: icmp_seq=1 ttl=64 time=0.049 ms
64 bytes from 172.17.10.10: icmp_seq=2 ttl=64 time=0.062 ms

--- 172.17.10.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1047ms
rtt min/avg/max/mdev = 0.049/0.055/0.062/0.006 ms
```

```bash
root@localhost:/# ping 172.17.10.20 -c 2
PING 172.17.10.20 (172.17.10.20) 56(84) bytes of data.
64 bytes from 172.17.10.20: icmp_seq=1 ttl=64 time=0.068 ms
64 bytes from 172.17.10.20: icmp_seq=2 ttl=64 time=0.083 ms

--- 172.17.10.20 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1061ms
rtt min/avg/max/mdev = 0.068/0.075/0.083/0.007 ms
```

<br>

**验证网络连通性**：测试 **nginx 的 eth0** 与 **docker0 网桥**、**gitlab 的 eth0** 之间的通信是否正常。

```bash
root@localhost:~# ping 172.17.0.1 -c 2
PING 172.17.0.1 (172.17.0.1) 56(84) bytes of data.
64 bytes from 172.17.0.1: icmp_seq=1 ttl=64 time=0.075 ms
64 bytes from 172.17.0.1: icmp_seq=2 ttl=64 time=0.052 ms

--- 172.17.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1059ms
rtt min/avg/max/mdev = 0.052/0.063/0.075/0.011 ms
```

```bash
root@localhost:~# ping 172.17.10.20 -c 2
PING 172.17.10.20 (172.17.10.20) 56(84) bytes of data.
64 bytes from 172.17.10.20: icmp_seq=1 ttl=64 time=0.030 ms
64 bytes from 172.17.10.20: icmp_seq=2 ttl=64 time=0.048 ms

--- 172.17.10.20 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1036ms
rtt min/avg/max/mdev = 0.030/0.039/0.048/0.009 ms
```

<br>

**验证网络连通性**：测试 **gitlab 的 eth0** 与 **docker0 网桥**、**nginx 的 eth0** 之间的通信是否正常。

```bash
root@localhost:/mnt# ping 172.17.0.1 -c 2
PING 172.17.0.1 (172.17.0.1) 56(84) bytes of data.
64 bytes from 172.17.0.1: icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from 172.17.0.1: icmp_seq=2 ttl=64 time=0.053 ms

--- 172.17.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1023ms
rtt min/avg/max/mdev = 0.050/0.051/0.053/0.001 ms
```

```bash
root@localhost:/mnt# ping 172.17.10.10 -c 2
PING 172.17.10.10 (172.17.10.10) 56(84) bytes of data.
64 bytes from 172.17.10.10: icmp_seq=1 ttl=64 time=0.038 ms
64 bytes from 172.17.10.10: icmp_seq=2 ttl=64 time=0.085 ms

--- 172.17.10.10 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1030ms
rtt min/avg/max/mdev = 0.038/0.061/0.085/0.023 ms
```

<br>

**检查 Docker 网络命名空间**：查看 **iptables filter 表** 中 **FORWARD 链** 的默认规则配置。

```bash
root@localhost:/# iptables -t filter -L 
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         
```

检查 **iptables FORWARD 链** 的默认策略（policy）**为 ACCEPT**，若不是，则使用以下命令进行配置调整。

```bash
iptables -P FORWARD ACCEPT
```

<br>

虽然流量可以正常转发，但**源 IP 仍为 172.17.x.x（Docker 内部网络）**，导致外部网络无法正确响应。此时需要配置 **NAT（地址转换）** 以修正源 IP。

```bash
root@localhost:~# ping 8.8.8.8 -c 2
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1034ms
```

![image-20250416082618202](/assets/img/linux-advanced-networking/image-20250416082618202.png)

<br>

**配置 NAT 规则**：为 **172.17.0.0/16 网段** 启用 NAT，确保 **Nginx 和 GitLab 容器** 的网络命名空间能够正常访问外部网络。

```
iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

<br>

**网络连通性测试**：验证 **Nginx 网络命名空间** 与 **外部网络** 的通信是否正常。

```bash
root@localhost:~# ping 8.8.8.8 -c 2
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=48 time=122 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=48 time=142 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 121.506/131.977/142.448/10.471 ms
```

![image-20250416082809914](/assets/img/linux-advanced-networking/image-20250416082809914.png)

<br>

**网络连通性测试**：验证 **Gitlab 网络命名空间** 与 **外部网络** 的通信是否正常。

```bash
root@localhost:~# ping 8.8.8.8 -c 2
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=48 time=257 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=48 time=174 ms

--- 8.8.8.8 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 174.149/215.368/256.588/41.219 ms
```

![image-20250416082902737](/assets/img/linux-advanced-networking/image-20250416082902737.png)

<br>

**配置 DNAT 端口转发规则**

- 将 **8080 端口** 的流量转发至 **172.17.10.10（Nginx）** 的 **80 端口**
- 将 **9090 端口** 的流量转发至 **172.17.10.20（GitLab）** 的 **80 端口**

```bash
iptables -t nat -A PREROUTING -p tcp --dport 8080 -j DNAT --to-destination 172.17.10.10:80
iptables -t nat -A PREROUTING -p tcp --dport 9090 -j DNAT --to-destination 172.17.10.20:80
```

<br>

在 **Nginx** 和 **GitLab** 的网络命名空间中启动 **80 端口** 的 Web 服务

```bash
root@localhost:/opt# mkdir nginx gitlab
root@localhost:/opt# touch nginx/nginx.file
root@localhost:/opt# touch gitlab/gitlab.file

root@localhost:/opt# tree
.
├── gitlab
│   └── gitlab.file
└── nginx
    └── nginx.file
```

```bash
root@localhost:~# ip netns exec nginx python3 -m http.server 80 --bind 0.0.0.0 --directory /opt/nginx
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

```bash
root@localhost:~# ip netns exec gitlab python3 -m http.server 80 --bind 0.0.0.0 --directory /opt/gitlab
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
```

<br>

从外部网络通过浏览器访问 **192.168.54.230:8080**（对应 Nginx）

![image-20250416084733925](/assets/img/linux-advanced-networking/image-20250416084733925.png)

<br>

从外部网络通过浏览器访问 **192.168.54.230:9090**（对应 GitLab）

![image-20250416084710975](/assets/img/linux-advanced-networking/image-20250416084710975-1744764874132-1.png)

