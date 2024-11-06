
# 几种通过 iproute2 来打通不同节点间容器网络的方式


几种通过 iproute2 来打通不同节点间容器网络的方式


* [host\-gw](https://github.com)
* [ipip](https://github.com)
* [vxlan](https://github.com):[milou加速器](https://xinminxuehui.org)


## 背景


之前由于需要打通不同节点间的容器网络，对 **flannel** 进行修改增加了网络相关信息的获取逻辑，进而可以完整使用 *backend* 的功能。
最近想拿掉这个 **flannel**，所以分析一下 *backend* 的功能，使用 **ip** 命令来进行模拟


**flannel** 的 *backend* 提供了 7 种类型，这里只对以下 3 种内核提供的功能作模拟


* host\-gw
* ipip
* vxlan


### 环境


容器使用**网络命名空间**隔离网络相关资源，为了操作简单直接使用**网络命名空间**作为测试环境。


vm1(192\.168\.32\.245\) 和 vm2(192\.168\.32\.246\) 是物理机上的两个虚拟机，操作系统为 CentOS 7\.2。


**目标为将 172\.245\.0\.0/24 和 172\.246\.0\.0/24 网段打通。**



```
┌─────────────────────────────┐          ┌─────────────────────────────┐
│ vm1                         │          │ vm2                         │
│     ┌─────────────────┐     │          │     ┌─────────────────┐     │
│     │ ns245           │     │          │     │ ns246           │     │
│     │   172.245.0.2   │     │          │     │   172.246.0.2   │     │
│     │      eth0       │     │          │     │      eth0       │     │
│     └────────|────────┘     │          │     └────────|────────┘     │
│              │              │          │              │              │
│           veth245           │          │           veth246           │
│              │              │          │              │              │
│            br245            │          │            br246            │
│       172.245.0.1/24        │          │       172.246.0.1/24        │
│                             │          │                             │
│                             │          │                             │
│      192.168.32.245/24      │          │      192.168.32.246/24      │
│            eth0             │          │            eth0             │
└──────────────│──────────────┘          └──────────────│──────────────┘
               │                                        │
               └────────────────────────────────────────┘

```

### 环境配置


需要打开 **ip\_forward** 选项，让流量能够通过网桥进入命名空间



```
sysctl -w net.ipv4.ip_forward=1

```

设置网桥 vm1(192\.168\.32\.245\)



```
# 创建网桥
ip link add br245 type bridge

# 启用网桥
ip link set br245 up

```

创建命名空间、设置虚拟网卡 vm1(192\.168\.32\.245\)



```
# 创建网络命名空间
ip netns add ns245

# 创建 veth-peer，并且一端设置在 netns 中
ip link add veth245 type veth peer name eth0 netns ns245

# 启用 netns 中的 veth-peer
ip netns exec ns245 ip link set eth0 up

# 启用 host 中的 veth-peer
ip link set veth245 up

# 将 host 中的 veth-peer 挂载到 br245 网桥上
ip link set veth245 master br245

```

设置网卡地址和路由 vm1(192\.168\.32\.245\)



```
# 设置网桥地址
ip addr add 172.245.0.1/24 dev br245

# 设置命名空间内的 veth-peer 地址
ip netns exec ns245 ip addr add 172.245.0.2/24 dev eth0

# 设置命名空间内的默认路由（新建容器也会存在一条这个默认路由）
ip netns exec ns245 ip route add default via 172.245.0.1 dev eth0

```


> vm2(192\.168\.32\.246\) 中配置和 vm1(192\.168\.32\.245\) 操作相同，245 和 246 互换即可


设置完成后，vm1(192\.168\.32\.245\) 中的网卡、地址和路由信息如下（略去无关网卡）



```
$ ip addr
2: eth0:  mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 52:54:00:ce:51:a5 brd ff:ff:ff:ff:ff:ff
    inet 192.168.32.245/24 brd 192.168.32.255 scope global noprefixroute eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fece:51a5/64 scope link 
       valid_lft forever preferred_lft forever
9: br245:  mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether 02:8d:b1:91:fa:7a brd ff:ff:ff:ff:ff:ff
    inet6 fe80::7426:b5ff:fea9:d35/64 scope link 
       valid_lft forever preferred_lft forever
10: veth245@if2:  mtu 1500 qdisc noqueue master br245 state UP group default qlen 1000
    link/ether 02:8d:b1:91:fa:7a brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::8d:b1ff:fe91:fa7a/64 scope link 
       valid_lft forever preferred_lft forever

$ ip route
default via 192.168.32.1 dev eth0 proto static metric 100 
172.245.0.0/24 dev br245 proto kernel scope link src 172.245.0.1 
192.168.32.0/24 dev eth0 proto kernel scope link src 192.168.32.245 metric 100

$ ip netns exec ns245 ip addr
2: eth0@if10:  mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether be:9c:d4:57:58:5a brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.245.0.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::bc9c:d4ff:fe57:585a/64 scope link 
       valid_lft forever preferred_lft forever

$ ip netns exec ns245 ip route
default via 172.245.0.1 dev eth0 
172.245.0.0/24 dev eth0 proto kernel scope link src 172.245.0.2

```

## host\-gw


只需要在 vm 中设置一条指向对方的路由即可



```
# vm1 (192.168.32.245)
ip route add 172.246.0.0/24 via 192.168.32.246 dev eth0 onlink

# vm2 (192.168.32.246)
ip route add 172.245.0.0/24 via 192.168.32.245 dev eth0 onlink

```

在命名空间内测试 172\.246\.0\.2 是否能通



```
$ ip netns exec ns245 ping -c 1 172.246.0.2
PING 172.246.0.2 (172.246.0.2) 56(84) bytes of data.
64 bytes from 172.246.0.2: icmp_seq=1 ttl=62 time=0.656 ms

--- 172.246.0.2 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.656/0.656/0.656/0.000 ms

```

从 172\.245\.0\.2 访问 172\.246\.0\.2 的时序图如下（回包方向逻辑一致）



```
172.245.0.2  ns245:eth0   br245       vm1:eth0    vm2:eth0        br246    ns246:eth0    172.246.0.2
    |           |           |             |           |             |           |             |
    |  default  |           |             |           |             |           |             |
    |  routing  |           |             |           |             |           |             |
    | --------> |           |             |           |             |           |             |
    |           | veth-peer |             |           |             |           |             |
    |           |  master   |             |           |             |           |             |
    |           | --------> |             |           |             |           |             |
    |           |           | 172.246.0.2 |           |             |           |             |
    |           |           |   routing   |           |             |           |             |
    |           |           | ----------> |           |             |           |             |
    |           |           |             |  layer 2  |             |           |             |
    |           |           |             | --------> |             |           |             |
    |           |           |             |           | 172.246.0.2 |           |             |
    |           |           |             |           |   routing   |           |             |
    |           |           |             |           | ----------> |           |             |
    |           |           |             |           |             |  master   |             |
    |           |           |             |           |             | veth-peer |             |
    |           |           |             |           |             | --------> |             |
    |           |           |             |           |             |           | 172.246.0.2 |
    |           |           |             |           |             |           | ----------> |
    |           |           |             |           |             |           |             |
172.245.0.2  ns245:eth0   br245       vm1:eth0    vm2:eth0       br246     ns246:eth0    172.246.0.2

```

### Q\&A


**Q**: 为何 **host\-gw** 模式下的容器所在宿主机必须在同一网段（二层互通）
**A**: 路由不会修改源IP，回包交换源目的 IP 后，网关查询路由表不能确定发包接口，二层互通不存在这个问题，不需要查询路由表，直接根据 MAC 地址选路


## ipip


IPIP（IP\-in\-IP）协议是一种网络层隧道协议，用于在一个IP网络上传输另一个IP数据包。IPIP协议的主要目的是在网络之间提供透明的通信通道，使得内部网络的数据包可以通过外部网络传输，而不需要改变数据包的内容。


协议格式如下



```
// 原始 TCP 数据包                
+--------------------------------+
|              ....              |
+--------------------------------+
|               TCP              |
+--------------------------------+
|               IP               |
+--------------------------------+
|            Ethernet            |
+--------------------------------+

// ipip 封装后的数据包
+--------------------------------+
|               ....             |
+--------------------------------+
|               TCP              |
+--------------------------------+
|               IP               |
+--------------------------------+
/           IP (tunnel)          /
+--------------------------------+
|            Ethernet            |
+--------------------------------+

```

在 vm1 (192\.168\.32\.245\) 中设置 **ipip** 隧道



```
# 创建 IPIP 隧道接口
ip tunnel add tun245 mode ipip local 192.168.32.245 remote 192.168.32.246

# 配置 IPIP 隧道接口的 IP 地址
ip addr add 172.245.1.1/30 dev tun245

# 启用 IPIP 隧道接口
ip link set tun245 up

# 添加路由，使 172.246.0.0/24 网段的数据包通过 IPIP 隧道传输
ip route add 172.246.0.0/24 via 172.245.1.2 dev tun245

```


> vm2(192\.168\.32\.246\) 中配置和 vm1(192\.168\.32\.245\) 操作相同，245 和 246 互换即可


在 vm2(192\.168\.32\.246\) 中访问 *172\.245\.0\.2* （增加访问 *192\.168\.32\.245* 进行对比）



```
$ ping -c 1 192.168.32.245
$ ip netns exec ns246 ping -c 1 172.245.0.2

```

在 vm1(192\.168\.32\.245\) 中对 *eth0* 进行抓包



```
$ tcpdump -s0 -i eth0 host 192.168.32.246  -nn
11:28:01.268176 IP 192.168.32.246 > 192.168.32.245: ICMP echo request, id 1730, seq 1, length 64
11:28:01.268299 IP 192.168.32.245 > 192.168.32.246: ICMP echo reply, id 1730, seq 1, length 64
11:28:08.697558 IP 192.168.32.246 > 192.168.32.245: IP 172.246.0.2 > 172.245.0.2: ICMP echo request, id 1708, seq 26, length 64 (ipip-proto-4)
11:28:08.697760 IP 192.168.32.245 > 192.168.32.246: IP 172.245.0.2 > 172.246.0.2: ICMP echo reply, id 1708, seq 26, length 64 (ipip-proto-4)

```

在 wireshark 更直观的看到 **ipip** 多了一层 ip，用来作为隧道通信。



```
Frame 3: 118 bytes on wire (944 bits), 118 bytes captured (944 bits)
Ethernet II, Src: 52:54:00:b9:5d:53 (52:54:00:b9:5d:53), Dst: 52:54:00:ce:51:a5 (52:54:00:ce:51:a5)
Internet Protocol Version 4, Src: 192.168.32.246, Dst: 192.168.32.245
Internet Protocol Version 4, Src: 172.246.0.2, Dst: 172.245.0.2
Internet Control Message Protocol

```

从 172\.245\.0\.2 访问 172\.246\.0\.2 的时序图如下（省略命名空间内的部分，回包方向逻辑一致）



```
br245        tun245      vm1:eth0    vm2:eth0      tun246         br246
  |             |           |           |             |             |
  | 172.246.0.2 |           |           |             |             |
  |  routing    |           |           |             |             |
  | ----------> |           |           |             |             |
  |             | ipip pack |           |             |             |
  |             | --------> |           |             |             |
  |             |           |  layer 2  |             |             |
  |             |           | --------> |             |             |
  |             |           |           | ipip unpack |             |
  |             |           |           | ----------> |             |
  |             |           |           |             | 172.246.0.2 |
  |             |           |           |             |   routing   |
  |             |           |           |             | ----------> |
  |             |           |           |             |             |
br245        tun245      vm1:eth0    vm2:eth0      tun246         br246

```

### Q\&A


**Q**: 和 **host\-gw** 最大的区别是什么
**A**: 对容器（网络命名空间）所在节点的网络不再限制二层互通，只要节点网络（二层、三层）互通即可。


## vxlan


**vxlan** 旨在解决大型云数据中心和多租户环境中传统 vlan（虚拟局域网）技术的局限性。通过在 UDP 之上封装第二层以太网帧，实现在第三层（IP）网络上的二层网络扩展，从而允许创建多达 1600 万个隔离的虚拟网络，远超 vlan 的4096 个网络限制。


协议格式如下



```
// 原始 TCP 数据包
+----------------------------------------+
|                  ....                  |
+----------------------------------------+
|                   TCP                  |
+----------------------------------------+
|                   IP                   |
+----------------------------------------+
|                Ethernet                |
+----------------------------------------+


// vxlan 封装后的数据包
+----------------------------------------+
|                  ....                  |
+----------------------------------------+
|                   TCP                  |
+----------------------------------------+
|                   IP                   |
+----------------------------------------+
|                Ethernet                |
+----------------------------------------+
/          VXLAN Header (8 bytes)        /
+----------------------------------------+
/               UDP (tunnel)             /
+----------------------------------------+
/               IP (tunnel)              /
+----------------------------------------+
/              Ethernet (tunnel)         /
+----------------------------------------+

```

在 vm1 中设置



```
# 创建 VXLAN 隧道接口
ip link add vtep245 type vxlan id 1 local 192.168.32.245 dev eth0 dstport 8472 nolearning

# 配置 IP
ip addr add 172.245.0.0/32 dev vtep245

# 设置 MTU
ip link set vtep245 mtu 1450

# 启动 vtep
ip link set veth245 up

# 配置路由
ip route add 172.246.0.0/24 via 172.246.0.0 dev vtep245 onlink

```


> vm2(192\.168\.32\.246\) 中配置和 vm1(192\.168\.32\.245\) 操作相同，245 和 246 互换即可


查看虚拟网卡，看到设置的 vxlan id、网卡和本地 ip



```
root@vm1# ip -d link show vtep245
12: vtep245:  mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 52:3a:8c:c9:08:e8 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 1 local 192.168.32.245 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535

root@vm2# ip -d link show vtep246
10: vtep246:  mtu 1450 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 02:38:4a:12:3b:43 brd ff:ff:ff:ff:ff:ff promiscuity 0 
    vxlan id 1 local 192.168.32.246 dev eth0 srcport 0 0 dstport 8472 nolearning ageing 300 noudpcsum noudp6zerocsumtx noudp6zerocsumrx addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535

```

查看监听 UDP 端口，看到内核监听了 **8742**



```
$ netstat -nulp
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
udp        0      0 0.0.0.0:8472            0.0.0.0:*                           -  

```

当前的网络拓扑图如下



```
┌─────────────────────────────┐          ┌─────────────────────────────┐
│ vm1                         │          │ vm2                         │
│     ┌─────────────────┐     │          │     ┌─────────────────┐     │
│     │ ns245           │     │          │     │ ns246           │     │
│     │   172.245.0.2   │     │          │     │   172.246.0.2   │     │
│     │      eth0       │     │          │     │      eth0       │     │
│     └────────|────────┘     │          │     └────────|────────┘     │
│              │              │          │              │              │
│           veth245           │          │           veth246           │
│              │              │          │              │              │
│            br245            │          │            br246            │
│       172.245.0.1/24        │          │       172.246.0.1/24        │
│                             │          │                             │
│           vtep245           │          │           vtep246           │
│      vni:1 172.245.0.0      │          │      vni:1 172.246.0.0      │
│                             │          │                             │
│      192.168.32.245/24      │          │      192.168.32.246/24      │
│            eth0             │          │            eth0             │
└──────────────│──────────────┘          └──────────────│──────────────┘
               │                                        │
               └────────────────────────────────────────┘

```

在 vm1(192\.168\.32\.245\) 上尝试通过 172\.245\.0\.2 访问 172\.246\.0\.2，无法访问，通过 tcpdump 诊断流量到达虚拟网卡 *vtep245*，但是不知道网关 *172\.246\.0\.0* 的 MAC 地址



```
$ ip netns exec ns245 ping -c 1 172.246.0.2

$ tcpdump -s0 -i vtep245 -nn
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
17:44:26.916277 ARP, Request who-has 172.246.0.0 tell 172.245.0.0, length 28

```

在上面已经知道了网关的 MAC 地址，直接进行手动设置（flannel 中有 etcd 进行集中管理）



```
# vm1
ip neigh add 172.246.0.0 dev vtep245 lladdr 02:38:4a:12:3b:43

# vm2
ip neigh add 172.245.0.0 dev vtep246 lladdr 52:3a:8c:c9:08:e8

```

解决 MAC 地址问题后，还有一个问题：**vxlan** 封的包如何发送出去，这里使用 *fdb* 来根据 MAC 指定 **vxlan** 包发送的地址



> fdb 是一个用于存储 MAC 地址及其对应端口的信息表，以便桥接设备能够高效地转发数据帧。



```
# vm1
bridge fdb add 02:38:4a:12:3b:43 dev vtep245 dst 192.168.32.246

# vm2
bridge fdb add 52:3a:8c:c9:08:e8 dev vtep246 dst 192.168.32.245

```

从 172\.245\.0\.2 访问 172\.246\.0\.2 的时序图如下（省略命名空间内的部分，回包方向逻辑一致）



```
br245        vtep245      vm1:eth0    vm2:eth0      vtep246         br246
  |             |            |           |              |             |
  | 172.246.0.2 |            |           |              |             |
  |  routing    |            |           |              |             |
  | ----------> |            |           |              |             |
  |             | vxlan pack |           |              |             |
  |             | ---------> |           |              |             |
  |             |            |  udp 8742 |              |             |
  |             |            | --------> |              |             |
  |             |            |           | vxlan unpack |             |
  |             |            |           | -----------> |             |
  |             |            |           |              | 172.246.0.2 |
  |             |            |           |              |   routing   |
  |             |            |           |              | ----------> |
  |             |            |           |              |             |
br245        vtep245      vm1:eth0    vm2:eth0      vtep246         br246

```

### Q\&A


**Q**: 为何 vm1 上的路由要使用 *172\.246\.0\.0* 作为网关地址
**A**:


* 网关不能使用本机的 IP 或者处于本机网段内的 IP，没有路由的情况下，流量转发不到外部；
* *172\.246\.0\.0* 不是一个可用的地址，用来标识整个网络，使用其他的地址会有冲突；


## 总结


从功能上看：


1. **host\-gw** 最简单，只需要一条路由就可以打通不同节点的容器网络，但只能在二层互通的场景下完成
2. **ipip** 在 **host\-gw** 的基础上建立了一条 IP 隧道，这样在三层互通的情况下也可以打通容器网络
3. **vxlan** 在类似 **ipip** 的基础上，基于三层隧道实现了二层的覆盖网络，提供了极高的网络隔离能力，相对也最复杂


从性能上看：


1. **host\-gw** 最快，因为没有任何封包的消耗
2. **ipip** 次之，多了二十字节的 IP 头的封装
3. **vxlan** 相对最慢，多了一整个 以太网、IP 层和 UDP 层的封装


## 参考


1. [https://www.rfc\-editor.org/rfc/inline\-errata/rfc7348\.html](https://github.com), Virtual eXtensible Local Area Network (VXLAN): A Framework for Overlaying Virtualized Layer 2 Networks over Layer 3 Networks


