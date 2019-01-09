---
description: 问题 
---

# macvlan 遇到的问题

首先k8s 集群使用的是macvlan cni 插件， 默认是macvlan 的bridge， macvlan 有四种模式， bridge 是相对高效并且比较适合k8s 网络互联的场景。

## 问题1

给主机网卡em2 配置macvlan 给容器， 并且mavclan 会封装到容器的net Namespace中

最初的macvlan 的cni 配置如下:
```
{
	"name": "mynet",
	"type": "macvlan",
	"master": "em2",
	"mode": "bridge",
	"ipam": {
		"type": "host-local",
		"ranges": [
			[
				{
					"subnet": "172.20.7.0/24",
					"rangeStart": "172.20.7.51",
					"rangeEnd": "172.20.7.98",
					"gateway": "172.20.7.254"
				}
			]
		],
		"routes": [
			{ "dst": "0.0.0.0/0"},
		]
	}
}

``` 

配置好后， 创建的容器可以与外面的主机互联， 并且同一个macvlan下的容器也能互相访问（ping 通） ,此时主机的ip地址是配置在em2 上， 这就导致容器不能访问本主机， 本主机也不能访问容器。

这是macvlan 本身的作用

若想让容器访问本主机， 需要给主机网卡em2 也配置一个macvlan，并且把原来主机em2 上的ip 转移到这个macvlan上， 这样才能实现互通。

## 问题2

在容器内， telnet 10.32.0.1 443 不通， 首先我们需要让容器内有一条路由， 走10.32.0.0 网络， 因此修改cni配置, 加一条路由
```
{
	"name": "mynet",
	"type": "macvlan",
	"master": "em2",
	"mode": "bridge",
	"ipam": {
		"type": "host-local",
		"ranges": [
			[
				{
					"subnet": "172.20.7.0/24",
					"rangeStart": "172.20.7.51",
					"rangeEnd": "172.20.7.98",
					"gateway": "172.20.7.254"
				}
			]
		],
		"routes": [
			{ "dst": "0.0.0.0/0"},
			{ "dst": "10.32.0.0/24", "gw": "172.20.7.100"}
		]
	}
}
```

默认macvlan 的数据包是不走主机的iptables的。


这样加上路由后， tcpdump后， 发现数据包能出去，但是没有返回， 出现的现象为
```
导致telnet 出现 No route to host 错误， 

tcpdump抓包中会出现 ICMP host 10.32.0.1 unreachable

```
unreachable 很可能是icmp 协议的问题

通过查看iptables 规则 发现，filter 表中， INPUT 链， 有对icmp-host-prohibited 的reject

```
REJECT     all  --  *      *       0.0.0.0/0            0.0.0.0/0            reject-with icmp-host-prohibited
```

删掉这条规则， 即可


## 问题三

配置的macvlan 和ip 在主机重启或者systemctl restart network 后， 原有的设置会消失

因此需要书写对macvlan的初始化脚本

参见https://github.com/kadisi/init-macvlan

注意： centos 在/etc/sysconfig/network-scripts 下的执行脚本没有默认对macvlan的设置， 我们需要自己写ifup-macvlan 和ifdown-macvlan


## 问题4

iptable 删掉那条reject 规则后，主机重启规则还会存在。
我们先需要stop 和disable 掉firewalld， 
```
systemctl disable firewalld
systemctl stop firewalld

yum install -y iptables-services

systemctl enable iptables

systemctl start iptables

```

iptables service 会保存iptables 规则到 /etc/sysconfig/iptables 文件下

## 问题5 关于  /etc/sysconfig/network-scripts 各个脚本的执行顺序

首先对于centos 用户空间的第一个init 程序
```
	centos5: SysV init 配置文件  /etc/inittab
	centos6: Upstart   配置文件  /etc/inittab;/etc/init/*.conf(主要
	cento7: systemd  配置文件 配置文件：/etc/systemd/system;/usr/lib/systemd/system

```

systemd 会启动network 服务 ,通过status 可以看到network 服务脚本
```
systemctl status network


● network.service - LSB: Bring up/down networking
   Loaded: loaded (/etc/rc.d/init.d/network; bad; vendor preset: disabled)
   Active: active (exited) since 一 2018-12-10 18:57:09 CST; 24min ago
     Docs: man:systemd-sysv-generator(8)
  Process: 1354 ExecStart=/etc/rc.d/init.d/network start (code=exited, status=0/SUCCESS)
    Tasks: 0
   Memory: 0B
```

```
cat /etc/rc.d/init.d/network

```
在上述脚本中， 你可以看到几个重要逻辑

```
# 这个就是获得/etc/sysconfig/network-scripts 下ifcfg-开头的各个文件
interfaces=$(ls ifcfg-* | \
        LC_ALL=C sed -e "$__sed_discard_ignored_files" \
               -e '/\(ifcfg-lo$\|:\|ifcfg-.*-range\)/d' \
               -e '{ s/^ifcfg-//g;s/[0-9]/ &/}' | \
        LC_ALL=C sort -k 1,1 -k 2n | \
        LC_ALL=C sed 's/ //')


start: 逻辑
    # bring up all other interfaces configured to come up at boot time
    for i in $interfaces; do
	
	... 省略

    action $"Bringing up interface $i: " ./ifup $i boot
        [ $? -ne 0 ] && rc=1
    done

```

start 逻辑实际上就是调用了 /etc/sysconfig/network-scripts 下的ifup 脚本， 对各个cfg 进行执行。


具体可以看一下ifup 的逻辑


# kube-proxy

kuber-proxy 有个 配置 `--cluster-cidr`, 这个配置参数代表的是kubernetes 集群的cluster cidr， 就是service ip， 设置好后， kube-proxy 会对源地址不是 cluster cidr 的数据报标记 masq， 自动做snat.

注意 kube-apiserver 也需要配置cluster cidr ,参数为  --service-cluster-ip-range=10.32.0.0/24， 这两个值要一致



