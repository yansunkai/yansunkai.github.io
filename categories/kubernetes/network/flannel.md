# Flannel
kubernetes默认集群中的容器都是直接连通的，而如何连通kubernetes没有具体实现，需要网络插件来实现容器跨主机连通，Flannel就是其中之一。  
Flannel有三种实现：VXLAN，HOST-GW，UDP。

## UDP模式
![flannel1](../../../image/kubernetes/flannel1.png) 
container1访问container2的数据包源地址：100.96.1.2 目标地址：100.96.2.3。  
根据容器内的路由规则，数据包来到docker0网桥，然后查看宿主机路由规则。  
flannel会在宿主机上维护下面一套路由规则  
```bash
# 在 Node 1 上
$ ip route
default via 10.168.0.1 dev eth0
100.96.0.0/16 dev flannel0  proto kernel  scope link  src 100.96.1.0
100.96.1.0/24 dev docker0  proto kernel  scope link  src 100.96.1.1
10.168.0.0/24 dev eth0  proto kernel  scope link  src 10.168.0.2
```
然后我们的数据包匹配第二条路由规则，发送给flannel0的设备。  
flannel0是个tunnel设备，tunnel设备会把数据包从内核态发送用户态的应用程序。  
这样数据包就交给flanneld进程处理。  
flanneld会维护一个Subnet，把集群中所有容器的subnet和宿主机关系存储在etcd中，然后查询etcd找到目标地址100.96.2.3对应
子网100.96.2.0/24的宿主机10.168.0.3，然后把数据包封包成源地址：10.168.0.2 目标地址：10.168.0.3的udp包，发送给8285端口，也就是
node2的flanneld进程，然后node2的flanneld进程解包，把里面的原始包发送给flannel0，然后根据路由最终到达container2。  
这里面的subnet可以通过docker的参数bid设置。  
* 这里面的udp丢包并没有关系因为是中间过程，丢包重试等通过外层的tcp等协议实现。  

因为udp的封包解包涉及到内核态和用户态的数据拷贝，所以效率比较低，性能较差，所以用vxlan模式代替。  

## Vxlan模式
VXLAN主要是通过VTEP（VXLAN Tunnel End Point）设备，所有连接到这个设备的docker或者vm都是二层连通的，就像在一个LAN里面。  
![flannel2](../../../image/kubernetes/flannel2.png) 