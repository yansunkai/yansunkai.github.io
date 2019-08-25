# CNI(Container Network Interface) 
CNI主要是kubernetes为不同网络插件提供一套接口框架方便网络插件开发。  
CNI主要定义了ADD和DEL两个事件，别发对应容器创建时候的网络配置，和容器销毁时的网络资源释放。

CNI插件其实就是/opt/cni/bin下执行网络配置所需的二进制文件
```bash
ls -al /opt/cni/bin/
total 73088
-rwxr-xr-x 1 root root  3890407 Aug 17  2017 bridge
-rwxr-xr-x 1 root root  9921982 Aug 17  2017 dhcp
-rwxr-xr-x 1 root root  2814104 Aug 17  2017 flannel
-rwxr-xr-x 1 root root  2991965 Aug 17  2017 host-local
-rwxr-xr-x 1 root root  3475802 Aug 17  2017 ipvlan
-rwxr-xr-x 1 root root  3026388 Aug 17  2017 loopback
-rwxr-xr-x 1 root root  3520724 Aug 17  2017 macvlan
-rwxr-xr-x 1 root root  3470464 Aug 17  2017 portmap
-rwxr-xr-x 1 root root  3877986 Aug 17  2017 ptp
-rwxr-xr-x 1 root root  2605279 Aug 17  2017 sample
-rwxr-xr-x 1 root root  2808402 Aug 17  2017 tuning
-rwxr-xr-x 1 root root  3475750 Aug 17  2017 vlan
```

## 主要工作原理
我们知道kubelet创建pod的时候会首先起一个infra容器，然后创建network namespace，
然后就会调用SetUpPod方法调用CNI插件的ADD方法。   
调用插件的具体参数可以参考[Link](https://github.com/containernetworking/cni/blob/master/SPEC.md#network-configuration)  
(plugin *cniNetworkPlugin) SetUpPod
```go
func (plugin *cniNetworkPlugin) SetUpPod(namespace string, name string, id kubecontainer.ContainerID, annotations, options map[string]string) error {
	if err := plugin.checkInitialized(); err != nil {
		return err
	}
	netnsPath, err := plugin.host.GetNetNS(id.ID)
	if err != nil {
		return fmt.Errorf("CNI failed to retrieve network namespace path: %v", err)
	}

	// Windows doesn't have loNetwork. It comes only with Linux
	if plugin.loNetwork != nil {
		if _, err = plugin.addToNetwork(plugin.loNetwork, name, namespace, id, netnsPath, annotations, options); err != nil {
			return err
		}
	}

	_, err = plugin.addToNetwork(plugin.getDefaultNetwork(), name, namespace, id, netnsPath, annotations, options)
	return err
}
```
(c *CNIConfig) AddNetworkList
```go
// AddNetworkList executes a sequence of plugins with the ADD command
func (c *CNIConfig) AddNetworkList(ctx context.Context, list *NetworkConfigList, rt *RuntimeConf) (types.Result, error) {
	var err error
	var result types.Result
	for _, net := range list.Plugins {
		result, err = c.addNetwork(ctx, list.Name, list.CNIVersion, net, result, rt)
		if err != nil {
			return nil, err
		}
	}

	if err = setCachedResult(result, list.Name, rt); err != nil {
		return nil, fmt.Errorf("failed to set network %q cached result: %v", list.Name, err)
	}

	return result, nil
}
```
CNI插件的具体实现根据不同类型插件做的事情也不一样。  
比如flannel会先创建CNI0网桥，然后VethPair，一段放到CNI0，配置IP等工作。
```bash
# 在宿主机上
$ ip link add cni0 type bridge
$ ip link set cni0 up

# 在容器里

# 创建一对 Veth Pair 设备。其中一个叫作 eth0，另一个叫作 vethb4963f3
$ ip link add eth0 type veth peer name vethb4963f3

# 启动 eth0 设备
$ ip link set eth0 up 

# 将 Veth Pair 设备的另一端（也就是 vethb4963f3 设备）放到宿主机（也就是 Host Namespace）里
$ ip link set vethb4963f3 netns $HOST_NS

# 通过 Host Namespace，启动宿主机上的 vethb4963f3 设备
$ ip netns exec $HOST_NS ip link set vethb4963f3 up 

# 在宿主机上
$ ip link set vethb4963f3 master cni0

# 在容器里
$ ip addr add 10.244.0.2/24 dev eth0
$ ip route add default via 10.244.0.1 dev eth0

# 在宿主机上
$ ip addr add 10.244.0.1/24 dev cni0
```

这里可以看到kubernetes其实是使用CNI插件来完成网络配置的，创建的网桥叫cni0而不是docker0，
做为容器的运行时，只要实现CRI就可以，所以不一定是docker。  