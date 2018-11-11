---
description: docker cgroup 
---

# cgroup 
namespace 解决的是环境隔离的问题， 这只是虚拟化中最最基础的一步，我们还需要解决对计算机资源使用上的隔离，也就是说虽然你通过Namespace 把我jail到一个特定的环境中去了，但是我在其中的进程使用CPU 内存 磁盘等这些计算资源其实还是可以随心所欲的。所以我们希望对进程进行资源利用上的限制或者控制。这就是Linux cGroup出来的原因。

Cgroup 可以让您为系统中所运行的任务进程的用户定义组群分配资源， 比如 CPU时间，系统内存，网络带宽或者这些资源的组合。 您可以监控您配置的cgroup 拒绝cgroup 访问某些资源，甚至在运行的系统中动态配置您的cgroup
cgroup提供了一下功能：

* Resource Limitation 限制资源使用 比如内存使用上限以及文件系统的缓存限制
* Priotitization 优先级控制，比如CPU利用和磁盘IO吞吐
* Accounting 一些审计或者控制，主要目的是为了计费
* Control 挂起进程，回复执行进程。

LInux 的cgroup这个事实已经实现成了一个文件系统， 您可以mount
您可以使用mount 命令查看
```
# mount -t cgroup
cgroup on /sys/fs/cgroup/cpuset type cgroup (rw,relatime,cpuset)
cgroup on /sys/fs/cgroup/cpu type cgroup (rw,relatime,cpu)
cgroup on /sys/fs/cgroup/cpuacct type cgroup (rw,relatime,cpuacct)
cgroup on /sys/fs/cgroup/memory type cgroup (rw,relatime,memory)
cgroup on /sys/fs/cgroup/devices type cgroup (rw,relatime,devices)
cgroup on /sys/fs/cgroup/freezer type cgroup (rw,relatime,freezer)
cgroup on /sys/fs/cgroup/blkio type cgroup (rw,relatime,blkio)
cgroup on /sys/fs/cgroup/net_prio type cgroup (rw,net_prio)
cgroup on /sys/fs/cgroup/net_cls type cgroup (rw,net_cls)
cgroup on /sys/fs/cgroup/perf_event type cgroup (rw,relatime,perf_event)
cgroup on /sys/fs/cgroup/hugetlb type cgroup (rw,relatime,hugetlb)
```






