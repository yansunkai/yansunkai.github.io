# 容器的资源限制（linux的Cgroups技术）  
Linux Cgroups全程：Linux Control Group，主要作用就是限制一个进程组能够使用的资源上线，包括CPU，内存，磁盘，网络带宽等。  

Cgroups暴露给用户的是在/sys/fs/cgroup路径下的文件系统。
mount -t cgroup可以查看。
![cgroup1](../../image/docker/docker-cgroups1.png)  
可以看到它由cpuset，spu，memory等子系统构成。

## 如何使用
### 我们以限制CPU资源举例：
首先进入/sys/fs/cgroup/cpu目录，创建container文件夹，你会发现系统自动生成对应的资源限制文件。  
![cgroup2](../../image/docker/docker-cgroups2.png)  