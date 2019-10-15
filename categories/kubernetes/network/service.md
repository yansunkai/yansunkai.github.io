# iptables模式
kubernetes内部主要利用service为pod提供固定ip和负载均衡。它的模式主要有iptables和ipvs两种。  

首先集群里有个kube-proxy组件它通过informer感知service对象的创建，然后根据service对象创建iptables规则。  

第一组规则：发往10.0.1.175:80的数据包会发往KUBE-SVC-NWV5X2332I4OT4T3这条规则。  
而10.0.1.175就是service的VIP。 可以看到这个ip只是iptables的规则上的一个配置并不是一个网络设备所以是ping不通的。  
```bash
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3
```

第二租规则：KUBE-SVC-NWV5X2332I4OT4T3是一组规则，它的数量和后端代理的pod数是一直的。而且通过设置probability字段实现pod的负责均衡。  
```bash
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR
```

第三组规则： 其实就是DNAT规则，修改目的地址为具体pod的地址。  
而另一条规则链是给数据包打上--set-xmark 0x00004000/0x00004000的标志，在后面的node pod模式访问服务的时候通过这个识别数据包然后进行SNAT。  
```bash
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376

```

# IPVS模式
IPVS模式工作原理是首先为service创建一个虚拟网卡比如kube-ipvs0，这个网卡的ip地址就是service的vip。  
```bash
# ip addr
  ...
  73：kube-ipvs0：<BROADCAST,NOARP>  mtu 1500 qdisc noop state DOWN qlen 1000
  link/ether  1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
  inet 10.0.1.175/32  scope global kube-ipvs0
  valid_lft forever  preferred_lft forever

```
然后kube-proxy设置ipvs模块为这个VIP设置三个IPVS虚拟主机，并且是通过轮询（RR）来负载均衡的。 而这个三台虚拟主机就是代理的后端pod。  
```bash
# ipvsadm -ln
 IP Virtual Server version 1.2.1 (size=4096)
  Prot LocalAddress:Port Scheduler Flags
    ->  RemoteAddress:Port           Forward  Weight ActiveConn InActConn     
  TCP  10.102.128.4:80 rr
    ->  10.244.3.6:9376    Masq    1       0          0         
    ->  10.244.1.7:9376    Masq    1       0          0
    ->  10.244.2.3:9376    Masq    1       0          0

```

一般集群我们会通过-proxy-mode=ipvs来开启ipvs模式，由于ipvs是通过内核的ipvs模块来实现的，不需要kube-proxy来维护转发的iptables规则所以性能比iptables模式好。