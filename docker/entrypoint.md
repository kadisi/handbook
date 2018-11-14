---
description: entrypoint cmd
---

# entrypoint cmd

大家在使用Docker 时候，经常对RUN， CMD, ENTRYPOINT 三个概念混淆。

RUN 是在build image 时候使用的命令， Dockerfile 中可以使用多条RUN命令，**并最终commit到镜像** 

CMD 和ENTRYPOINT 则是在运行container时候会运行的指令， 都只能写一条， 如果写多了，最后一条会生效。

CMD和ENTRYPOINT的区别是：

CMD 在运行时候，会被docker 的command 命令覆盖， ENTRYPOINT 不会被运行时的command 覆盖， 但是也可以指定

CMD 和ENTRYPOINT 一般用于制作具备后台服务的image，在使用image 启动container 时候，自动启动服务。 

同样执行 `docker run -it --rm <image_name> hello world`

如果是`ENTERYPOINT ["/bin/bash"]`那么实际运行的命令是 `/bin/bash hello world`

如果是`CMD ["/bin/bash"]`那么实际运行的命令是 `hello world`  
  
**即 运行容器时的命令在ENTRYPOINT时是作为ENTRYPOINT 的参数传递的， 在使用CMD 时候是直接替换CMD的。**

所以有一种取巧的用法，在 dockerfile 中同时使用二者:

```text
ENTRYPOINT ["mongod"]
CMD ["--help"]
```

这样用户不仅可以自定义启动 mongod 的参数，在不指定参数的时候还可以默认使用 --help 显示帮助信息



**CMD 和 ENTRYPOINT 运行形式都支持shell形式和exec形式**

shell形式： CMD  command param1 param2

exec形式： CMD \["executable", "param1", "param2"\]

另外CMD还多了一种为ENTRYPOINT提供参数的形式

CMD \["param1", "param2"\]



shell 形式和exec形式的本质区别在于shell形式 提供了默认指令 **/bin/sh -c** 

所以指定的command 将在shell环境下运行， 因此指定command 的pid 将不会是1， 因为pid 为1的是shell， command进程是shell的子进程

shell形式还有一个严重的问题：由于其默认使用/bin/sh来运行命令，如果镜像中不包含/bin/sh，容器会无法启动。

exec形式则不然， 其直接运行指定的指令：

```text
Mem: 323148K used, 1727468K free, 132628K shrd, 49964K buff, 149180K cached
CPU:  0.0% usr  0.0% sys  0.0% nic  100% idle  0.0% io  0.0% irq  0.0% sirq
Load average: 0.00 0.03 0.05 1/151 5
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
    1     0 root     R     1200  0.0   0  0.0 top -b
```

由于exec指定的命令不由shell启动，因此也就无法使用shell中的环境变量，如`$HOME`。如果希望能够使用环境变量，可以指定命令为sh：`CMD [ "sh", "-c", "echo", "$HOME" ]`







引用：

{% embed url="https://www.jianshu.com/p/1fd7d32f819a" %}

