# Table of contents

* [手记](README.md)

## docker

* [entrypoint cmd](docker/entrypoint.md)
* [namespace](docker/namespace.md)
* [cgroup](docker/cgroup.md)
* [flannel](docker/flannel.md)

## compute

* [操作系统](compute/os/README.md)
  * [锁](compute/os/lock.md)
  * [线程](compute/os/thread.md)
  * [总线锁](compute/os/buslock.md)
  * [总线](compute/os/bus.md)
  * [共享内存](compute/os/sharemem.md)
  * [共享库](compute/os/sharelibrary.md)
  * [软连接](compute/os/ln.md)
  * [进程](compute/os/process.md)
  * [整数 浮点数](compute/os/float.md)
  * [三态门](compute/os/tristate.md)
  * [信号](compute/os/signal.md)
  * [用户态/内核态](compute/os/use_kernal_mode.md)
  * [系统启动顺序](compute/os/os_start.md)
* [网络](compute/network/README.md)
  * [udp](compute/network/udp.md)
  * [tcp](compute/network/tcp.md)
  * [http](compute/network/http.md)
  * [tcp 引用](compute/network/tcp-xie-yi.md)
  * [http connect](compute/network/http-connect.md)
  * [dpdk](compute/network/dpdk.md)
  * [socket 套接字](compute/network/socket.md)
  * [select和poll](compute/network/select.md)
  * [iptables](compute/network/iptables.md)
  * [macvlan](compute/network/macvlan.md)
  * [arp](compute/network/arp.md)
  * [vlan](compute/network/vlan.md)
  * [vxlan](compute/network/vxlan.md)
  * [ip 数据包](compute/network/ip.md)
  * [路由](compute/network/route.md)
  * [广播和多播](compute/network/broad_multi_cast.md)
  * [socat](compute/network/socat.md)
* [编码](compute/encoding/README.md)
  * [ascii](compute/encoding/ascii_unicode_utf-8.md)

## algorithm

* [数据结构](algorithm/datastruct/README.md)
  * [链表](algorithm/datastruct/list.md)
  * [递归](algorithm/datastruct/recursion.md)
* [算法思想](algorithm/thought/README.md)
  * [二分](algorithm/thought/binary.md)
* [排序](algorithm/sort/README.md)
  * [堆排序](algorithm/sort/heap-sort.md)
  * [归并](algorithm/sort/merge-sort.md)
  * [线性排序](algorithm/sort/line-sort.md)
  * [其他](algorithm/sort/other-sort.md)
  * [快排](algorithm/sort/quick.md)
* [其他](algorithm/other/README.md)
  * [算术运算](algorithm/other/arithmetic.md)
  * [全排列](algorithm/other/fullarrange.md)

## language

* [golang](language/golang/README.md)
  * [基础](language/golang/base.md)
  * [垃圾回收](language/golang/gc.md)
  * [内存管理](language/golang/golang-memory-manager.md)
  * [unsafe.Pointer](language/golang/unsafe.pointer.md)
  * [slice](language/golang/slice.md)
  * [string](language/golang/string.md)
  * [reflect](language/golang/reflect.md)
  * [iota](language/golang/iota.md)
  * [接口](language/golang/interface.md)
  * [接口1](language/golang/interface-1.md)
  * [sync.RWMutex](language/golang/sync.rwmutex.md)
  * [lock](language/golang/lock.md)
  * [select 和switch](language/golang/select-he-switch.md)
  * [channel](language/golang/channel.md)
  * [数据争用](language/golang/dateraces.md)
  * [goroutine](language/golang/goroutine.md)

## shell

* [shell](shell/shell/README.md)
  * [if语法](shell/shell/grammar.md)
  * [eval](shell/shell/eval.md)
  * [string](shell/shell/string.md)

## work

* [问题](work/wen-ti.md)

## code analyze

* [kubernetes](code-analyze/kubernetes/README.md)
  * [borg](code-analyze/kubernetes/borg.md)
  * [kube-apiserver](code-analyze/kubernetes/apiserver.md)
  * [kube-controller-manager](code-analyze/kubernetes/kube-controller-manager/README.md)
    * [worke queue](code-analyze/kubernetes/kube-controller-manager/worke-queue.md)
    * [rs controller](code-analyze/kubernetes/kube-controller-manager/rs-controller.md)
    * [deploy controller](code-analyze/kubernetes/kube-controller-manager/deploy-controller.md)
  * [kube-scheduler](code-analyze/kubernetes/kube-scheduler/README.md)
    * [调度机制](code-analyze/kubernetes/kube-scheduler/tiao-du-ji-zhi.md)
    * [调度选举](code-analyze/kubernetes/kube-scheduler/tiao-du-xuan-ju.md)
    * [informerfactory 机制](code-analyze/kubernetes/kube-scheduler/informerfactory-ji-zhi.md)
  * [kubelet](code-analyze/kubernetes/kubelet/README.md)
    * [pod merge](code-analyze/kubernetes/kubelet/pod-merge.md)
* [containerd](code-analyze/containerd/README.md)
  * [深入理解kubernetes](code-analyze/containerd/deepintokubernetes/README.md)
    * [理解1](code-analyze/containerd/deepintokubernetes/deep1.md)
    * [理解2](code-analyze/containerd/deepintokubernetes/deep2.md)
    * [理解3](code-analyze/containerd/deepintokubernetes/deep3.md)
    * [参考](code-analyze/containerd/deepintokubernetes/reference.md)
  * [snapshot service](code-analyze/containerd/snapshot-service.md)
  * [core 代码](code-analyze/containerd/core-dai-ma.md)
  * [image spec](code-analyze/containerd/image-spec.md)
  * [image pull](code-analyze/containerd/image-pull.md)
  * [containerd-shim](code-analyze/containerd/containerd-shim.md)

