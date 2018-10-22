# flannel



  
flannel host-gw 模式， 实际上走的是交换机的二层，每个主机上都有其他相邻主机的路由，网关就是对方相邻主机。

 但是默认flannel 会设置docker 的ip 伪装设置，ip 伪装设置默认为true，

这样，从docker0 出来的数据包源ip是不会变的， 但是从docker0 -&gt; 物理网卡后，源ip会变成物理网卡的IP

根本原因是因为iptables nat 表中有一条规则： MASQUERADE,    

```go
MASQUERADE   
This target is only valid in the nat table, in the POSTROUTING chain.  It should only be used with dynamically assigned IP (dialup) connections: if you have a static IP address, you should use the SNAT  target.   Mas      querading  is  equivalent  to  specifying a mapping to the IP address of the interface the packet is going out, but also has the effect that connections are forgotten when the interface goes down.  This is the correct      behavior when the next dialup is unlikely to have the same interface address (and hence any established connections are lost anyway). 
```





 docker 默认是开启的， 如果不想用nat ，需要把她关掉。

