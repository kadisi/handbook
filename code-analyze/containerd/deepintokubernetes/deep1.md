---
description: 深入理解Kubernetes   容器运行时。。。
---

# 理解1

## 前言

随着容器技术的快速发展，生态逐渐地从单体化运行容器应用，发展为支持大规模容器编排，容器成为了进程执行单元。其中容器编排以Kubernetes,Mesos为代表。面对这些竞争，2016年6月，Docker宣布在Docker Engine中内置Swarm，极大简化了容器编排的复杂性。Google发起CRI（Container RuntimeInterface容器运行时接口）项目，通过shim的抽象层使得调度框架支持不同的容器引擎实现。

面对这些挑战，2016年12月，Docker宣布将Docker Engine的核心组件Containerd捐赠到一个新的开源社区独立发展和运营，目标是提供一个标准化的容器运行时，注重简单、 健壮性和可移植性。由于Containerd只包含容器管理最基本的能力，上层框架可以有更大的灵活性来提供容器的调度和编排能力。

所以Containerd是容器技术标准化之后的产物，为了能够兼容OCI标准，将容器运行时及其管理功能从Docker Daemon剥离。理论上，即使不运行dockerd，也能够直接通过Containerd来管理容器。（当然，Containerd本身也只是一个守护进程，容器的实际运行时由后面介绍的runC控制。）

Containerd向上为Docker Daemon提供了gRPC接口，使得Docker Daemon屏蔽下面的结构变化，确保原有接口向下兼容。向下通过Containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题。 1.10以后，Containerd 集成Cri-containerd插件，支持Kubelet CRI规范，可以让Kubelet 直接调用。

## Containerd 源码结构和编译步骤

Containerd 的源码现在托管在GitHub上， 地址为[https://github.com/containerd/containerd](https://github.com/containerd/containerd)。

本章节后续所有的代码分析都假设是Linux 、go1.10.2 ，Containerd 的代码分支 release/1.1 commit 号为 57508dcb0b5776efaacd0828ed42f819fab5ba07 的环境。

Containerd的编译命令很简单，只需要在代码的根目录下执行 make 即可，最后make命令会在bin目录下编译出对应的二进制:

```bash
mkdir -p $GOPATH/src/github.com/containerd
cd  $GOPATH/src/github.com/containerd
git clone git@github.com:containerd/containerd.git
# 或者 使用go get 命令 下载Containerd 源码: go get github.com/containerd/containerd 
cd containerd
make
# 输出结果
#🇩 bin/ctr
#🇩 bin/containerd
#🇩 bin/containerd-stress
#🇩 bin/containerd-release
#🇩 bin/containerd-shim
#🇩 bin/containerd-shim-runc-v1
#🇩 binaries

#若想编译单个二进制，可以执行make bin/{binaries name}, 例如单独编译containerd
make bin/containerd

#若想成为Containerd 社区的Contributer, 每次push 代码前，最好需要执行check 和 test 命令
#分别用来做代码检查和单元测试
# make check 需要提前预装 gometalinter 命令
make check
make test
```

若想成为Containerd 社区的Contributer，每次push 代码前，需要执行check 和test 命令，分别用来做代码检查和单元测试

```text
# make check 需要提前预装 gometalinter 命令
make check
make test
```

鉴于本章节只是对Containerd的代码进行分析，接下来的介绍如何在Mac 的Goland 开发环境中完成对Containerd的代码分析。Mac 下安装GoLand 2018.2.2 版本， 打开Goland -&gt; Open Project -&gt;选择在GOPATH下刚刚clone下的Containerd 目录 -&gt; Open 按钮。 即打开了Containerd 项目。

![](../../../.gitbook/assets/image%20%286%29.png)

由于我们只是在Mac 下查看Containerd 的代码，并不需要在Mac下编译，最终的运行环境是Linux，所以我们只关心跟Linux 平台相关的代码， 因此需要对Goland进行设置，打开Goland的Preferences -&gt; Go -&gt; Vendoring & Build Tag。 OS 选择linux，Arch 选择Default \(amd64\)

![](../../../.gitbook/assets/image%20%2823%29.png)

至此我们完成了对Goland 的设置，当然你可以设置golang 的主题，代码颜色，查看接口方法实现等快捷键，可以自行学习，使用Goland 建议充分使用Go TO -&gt; Implementation\(s\) 功能，这样能快速查看某个接口都有哪些结构体实现，方便代码追踪，默认快捷键为: Option+⌘+B。

下面的表格给出Containerd 主要的package 的源码分析结果

| Package | 描述 |
| :--- | :--- |
| api | api package定义了Containerd 提供的GRPC接口的相关proto描述文件和对应go 源码文件，例如Containerd  提供的container，task，image，content 接口都在api-services 目录下定义 |
| bin | 编译出的二进制文件目录 |
| cmd | 包括了Containerd所有后台进程的代码入口（例如containerd，containerd-shim） |
| containers |  |
| content |  |
| defaults | 默认参数 |
| events |  |
| filters | 各种参数的过滤操作 |
| image |  |
| labels |  |
| leases |  |
| metadata | metadata 数据相关处理逻辑，containerd 默认数据库使用的是boltdb |
| mount |  |
| namespaces |  |
| rumtime |  |
| server |  |
| services | 各个GRPC接口的具体实现 |
| snapshots |  |
| vendor/github.com/containerd/cri | cri plugin, Containerd 在1.1版本已经将Cri-containerd作为Plugin的形式对外提供服务，因此与kubelet集成时，已经不需要部署单独的Cri-containerd 服务。 |

## Containerd 架构

Containerd是一个遵循行业标准的容器运行时，它强调简单性，健壮性和可移植性。 可以管理其主机系统的完整容器生命周期，包括：映像传输和存储，容器执行，监视，存储和网络等。

Containerd涉及之初旨在嵌入到更大的系统中，例如Kubernetes，而不是由开发人员或最终用户直接使用。因此Containerd 对于最终用户而言在使用方面并不如Docker 那么友好。不过Containerd也提供ctr 命令行供测试和调试用。

![](../../../.gitbook/assets/image%20%2826%29.png)

Containerd 最上层提供一个最主要的GRPC 接口，供Docker 或者Kubelet 去调用， 第二层是各种资源对象，其中最主要的有Content，Snapshot，Images，Containers，Task 等资源对象， 其中metadata数据会存放到boltdb 本地数据库中， 而下载的Image manifest 等文件存放到本地特定目录下，最下层是Runtimes，Containerd 通过containerd-shim 默认调用runc 来实际创建容器。

![](../../../.gitbook/assets/image%20%289%29.png)

Containerd 在1.1版本已经将Cri-containerd作为Plugin的形式对外提供服务，即Containerd 代码中的 CRI Plugin， 因此与kubelet集成时，已经不需要部署单独的Cri-Containerd 服务。CRI Plugin 实现了image service 和 runtime service 接口，当CRI Plugin 接受到kubelet CRI client 的gRPC请求后， 会创建一个client 连接自身的GRPC plugin 服务， 调用相关的container，task，和snapshots等接口。同时CRI plugin 还会调用CNI接口，来进行对Pod 网络设置。

Containerd 每次创建一个container 都会给对应的container 起一个containerd-shim 服务，Containerd-shim 对外提供rRPC 服务，社区也正准备让Containerd-shim 支持gRPC, Containerd-shim 的目的是向上对接Containerd 向下可以支持不同的OCI runtime 实现， 现在Containerd-shim 默认使用的runc，通过runc来最终创建container。每个container 分配单独的Containerd-shim 服务的好处是，防止Containerd-shim 服务挂掉后，导致Containerd 无法对其他的Container进行访问。

## ctr 进程源码分析

Containerd 源码中主要包括了Containerd，ctr，Containerd-shim 相关代码和逻辑，本章节主要讲解ctr 命令相关代码和逻辑， ctr 是Containerd 默认的cli ，根据GRPC 与Containerd 交互。 ctr 提供Container，images, namespaces, tasks, snapshots 等相关命令将允许您创建和管理使用containerd运行的容器。

ctr 命令如下:

```text
COMMANDS:
     plugins, plugin           provides information about containerd plugins
     version                   print the client and server versions
     containers, c, container  manage containers
     content                   manage content
     events, event             display containerd events
     images, image, i          manage images
     leases                    manage leases
     namespaces, namespace     manage namespaces
     pprof                     provide golang pprof outputs for containerd
     run                       run a container
     snapshots, snapshot       manage snapshots
     tasks, t, task            manage tasks
     install                   install a new package
     shim                      interact with a shim directly
     cri                       interact with cri plugin
     help, h                   Shows a list of commands or help for one command
GLOBAL OPTIONS:
   --debug                      enable debug output in logs
   --address value, -a value    address for containerd's GRPC server (default: "/run/containerd/containerd.sock")
   --timeout value              total timeout for ctr commands (default: 0s)
   --connect-timeout value      timeout for connecting to containerd (default: 0s)
   --namespace value, -n value  namespace to use with commands (default: "default") [$CONTAINERD_NAMESPACE]
   --help, -h                   show help
   --version, -v                print the version
```

刚接触Containerd 时，可以先从ctr命令入手，学习如何创建、启动和进入容器以及如何拉取，导出镜像等等，在学习的过程中体会与Docker 命令操作的区别。

### ctr进程启动过程

ctr 进程的入口源码如下 ，相关注释已经在代码中。

```go
# 入口源码文件  cmd/ctr/main.go
# 入口main()函数
func main() {
    // 注释: 核心代码，app.New 里定义了不同COMMANDS 的执行逻辑， 可以着重了解
    app := app.New()
    app.Commands = append(app.Commands, pluginCmds...)
    // 注释: 核心代码，Run 会对ctr 命令参数进行解析，将相关参数和值传入到cli.Context 变量中，方法最终调用对应的子COMMAND逻辑,可以暂时略过
    if err := app.Run(os.Args); err != nil {
        fmt.Fprintf(os.Stderr, "ctr: %s\n", err)
        os.Exit(1)
    }
}
```

在app.New\(\)方法里我们可以看到不同子COMMAND的定义：

```go
# app.New() cmd/ctr/app/main.go
// New returns a *cli.App instance.
func New() *cli.App {
    ... 部分代码已经省略
    app.Commands = append([]cli.Command{
        plugins.Command,
        versionCmd.Command,
        // 注释: 定义了对子COMMAND container 的实现逻辑
        containers.Command,
        content.Command,
        events.Command,
        images.Command,
        namespacesCmd.Command,
        pprof.Command,
        run.Command,
        snapshots.Command,
        tasks.Command,
    }, extraCmds...)
    ... 部分代码已经省略
    return app
}
```

我们以containers.Command 为例进行讲解， containers.Command 包含了 container 的create、delete、info、list、label 操作逻辑：

```go
# 源码文件 cmd/ctr/commands/containers/containers.go
var Command = cli.Command{
    ... 部分代码已经省略
    Subcommands: []cli.Command{
        createCommand,
        deleteCommand,
        infoCommand,
        listCommand,
        setLabelsCommand,
    },
}
var createCommand = cli.Command{
    ... 部分代码已经省略
    Action: func(context *cli.Context) error {
        var (
            id  = context.Args().Get(1)
            ref = context.Args().First()
        )
        ... 部分代码已经省略
        // 注释: 创建gRPC client
        client, ctx, cancel, err := commands.NewClient(context)
        ... 部分代码已经省略
        defer cancel()
        // 注释: 调用NewContainer 创建container
        _, err = run.NewContainer(ctx, client, context)
        ... 部分代码已经省略
        return nil
    },
}
```

以createCommand 为例，ctr 会创建gRPC的client，之后会调用run.NewContainer 方法创建容器，NewContainer 方法实现如下：

```go
# 文件 cmd/ctr/commands/run/run_unix.go
func NewContainer(ctx gocontext.Context, client *containerd.Client, context *cli.Context) (containerd.Container, error) {
    ... 部分代码已经省略

    var (
        // 注释: Spec 结构体Opts 操作列表, Spec 是containers.Container 结构体的一个字段
        // 注释: Spec 的具体结构体的定义遵循了 runtime-spec的规范，以linux 为例，具体规范链接如下 
        // 注释: https://github.com/opencontainers/runtime-spec/blob/master/config-linux.md
        opts  []oci.SpecOpts
        // 注释: containers.Container 结构体Opts 操作列表
        cOpts []containerd.NewContainerOpts
        // 注释: 会根据opts变量最终给spec 变量的特定字段赋值
        spec  containerd.NewContainerOpts
    )
    opts = append(opts, oci.WithEnv(context.StringSlice("env")))
    opts = append(opts, withMounts(context))
    cOpts = append(cOpts, containerd.WithContainerLabels(commands.LabelArgs(context.StringSlice("label"))))
    cOpts = append(cOpts, containerd.WithRuntime(context.String("runtime"), nil))
    if context.Bool("rootfs") {
        opts = append(opts, oci.WithRootFSPath(ref))
    } else {
        // 注释: ctr命令中若没特定指定 默认是overlayfs
        snapshotter := context.String("snapshotter")
        // 注释: 通过GRPC调用Containerd 接口，查看此image 是否在Containerd的数据库中
        image, err := client.GetImage(ctx, ref)
        if err != nil {
            return nil, err
        }
        // 注释: 判断Image 是否完全下载下来，后续会详细介绍
        unpacked, err := image.IsUnpacked(ctx, snapshotter)
        if err != nil {
            return nil, err
        }
        if !unpacked {
            if err := image.Unpack(ctx, snapshotter); err != nil {
                return nil, err
            }
        }
        opts = append(opts, oci.WithImageConfig(image))
        cOpts = append(cOpts,
            containerd.WithImage(image),
            containerd.WithSnapshotter(snapshotter),
            // Even when "readonly" is set, we don't use KindView snapshot here. (#1495)
            // We pass writable snapshot to the OCI runtime, and the runtime remounts it as read-only,
            // after creating some mount points on demand.
            containerd.WithNewSnapshot(id, image))
    }
    ... 部分代码已经省略
    if context.IsSet("config") {
        var s specs.Spec
        if err := loadSpec(context.String("config"), &s); err != nil {
            return nil, err
        }
        spec = containerd.WithSpec(&s, opts...)
    } else {
        spec = containerd.WithNewSpec(opts...)
    }
    cOpts = append(cOpts, spec)

    // oci.WithImageConfig (WithUsername, WithUserID) depends on rootfs snapshot for resolving /etc/passwd.
    // So cOpts needs to have precedence over opts.
    // TODO: WithUsername, WithUserID should additionally support non-snapshot rootfs
    return client.NewContainer(ctx, id, cOpts...)
}
```

NewContainer\(\)方法在return之前，所做的主要任务就是组装cOpts 变量，cOpts 是 NewContainerOpts 列表，NewContainerOpts 类型定义在 container\_opts.go 文件中

```go
# 文件 /container_opts.go
// NewContainerOpts allows the caller to set additional options when creating a container
type NewContainerOpts func(ctx context.Context, client *Client, c *containers.Container) error

// WithImage sets the provided image as the base for the container
func WithImage(i Image) NewContainerOpts {
    return func(ctx context.Context, client *Client, c *containers.Container) error {
        c.Image = i.Name()
        return nil
    }
}
```

 NewContainerOpts 定义了一个方法，目的是对containers.Container 结构体的某些字段进行赋值。例如 上述的WithImage 方法，通过使用闭包的形式来设置containers.Container 变量c中Image 字段的值。在Containerd代码中，你会发现存在大量类似的代码风格，通过传入一组Opts的操作，使用for循环 一次性的设置某个结构体变量对应字段的值。

最终return client.NewContainer 方法会调用Containerd的GRPC接口创建Container，后续的代码分析中我们会了解到此调用方法仅仅是在Containerd 的boltdb数据库中，创建一条对应的Container数据，仅此而已，真正的Container 运行是在Task 对象中。

```go
# 文件 /client.go
// NewContainer will create a new container in container with the provided id
// the id must be unique within the namespace
func (c *Client) NewContainer(ctx context.Context, id string, opts ...NewContainerOpts) (Container, error) {
    ctx, done, err := c.WithLease(ctx)
    if err != nil {
        return nil, err
    }
    defer done(ctx)
    // 注释: 创建一个默认的container对象，id 为参数传进来的id
    container := containers.Container{
        ID: id,
        Runtime: containers.RuntimeInfo{
            Name: c.runtime,
        },
    }
    // 注释: 通过for 循环，利用opts 对container 对象对应字段进行赋值
    for _, o := range opts {
        if err := o(ctx, c, &container); err != nil {
            return nil, err
        }
    }
    // 注释: 调用ContainerService().Create()调用Containerd的GRPC接口
    // 注释: Create() 方法通过代码追踪可以看到由remoteContainers 类型实现
    // 注释: remoteContainers 在 /containerstore.go 定义
    r, err := c.ContainerService().Create(ctx, container)
    if err != nil {
        return nil, err
    }
    return containerFromRecord(c, r), nil
}
```

ctr 的代码逻辑相对简单，主要功能就是创建GRPC client 调用Containerd对应的GRPC接口，实现ctr 命令行对应的操作。 在后续的Containerd 进程源码分析中我们会继续从ctr 为入口进行分析。

## Containerd 进程源码分析

Containerd 进程是整个Containerd服务的核心模块，往上给kubelet，ctr，ctrctl 等提供GRPC接口，往下调用Containerd-shim 进程操作每个Container。

Containerd 的其他介绍 \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

在分析Containerd 之前我们需要提前了解一下相关的知识，这样对于我们快速了解Containerd会有很好的帮助

* **GRPC**:   google 开发，是一款语言中立、平台中立、开源的远程过程调用\(RPC\)系统
* **BoltDB** :：Containerd 进程默认的本地数据库为BoltDB以及数据在BoltDB中的存储格式  
* **runtime-spec** ：OCI 为容器指定的规范  
* **image-spec** ：OCI为Image 指定的规范 
* **配置文件**： Containerd 进程的配置文件
* **Plugins** ：Containerd 进程中各个Plugin的关系
* **root/state dir**： Containerd root/state 目录存放内容

### GRPC

在 gRPC 里客户端应用可以像调用本地对象一样直接调用另一台不同的机器上服务端应用的方法，使得您能够更容易地创建分布式应用和服务。与许多 RPC 系统类似，gRPC 也是基于以下理念：定义一个服务，指定其能够被远程调用的方法（包含参数和返回类型）。在服务端实现这个接口，并运行一个 gRPC 服务器来处理客户端调用。在客户端拥有一个存根能够像服务端一样的方法。

gRPC已经应用在Google的云服务和对外提供的API中，其主要应用场景如下：

* 低延迟、高扩展性、分布式的系统 
* 同云服务器进行通信的移动应用客户端 
* 设计语言独立、高效、精确的新协议 
* 便于各方面扩展的分层设计，如认证、负载均衡、日志记录、监控等

有兴趣的同学建议根据官方介绍，书写客户端和服务器端代码的例子，至少要学会客户端与服务器端四种交互模式（simpe，server-side streaming，client-side streaming，Bidirectional streaming ）这样能加深对Containerd源码的理解，官方链接：[https://grpc.io/](https://grpc.io/)

### BoltDB

BoltDB是一个由golang实现嵌入式key/value的数据库。BoltDB提供的API来高效的存取数据。而且支持完全可序列化的ACID事务，让应用程序可以更简单的处理复杂操作。BoltDB 接口中封装了比如：更新，查看，批量更新等事务的接口，方便开发人员直接使用。

```text
func (db *DB) Update(fn func(*Tx) error) error 
func (db *DB) View(fn func(*Tx) error) error 
func (db *DB) Batch(fn func(*Tx) error) error
```

建议在看Containerd 代码前，先对BoltDB 熟悉一遍，了解db 中bucket 的创建，嵌套查询，key/value 的更新查询等， 这让我们以后分析Containerd代码会更加顺畅。具体了解和使用请访问 [https://github.com/boltdb/bolt](https://github.com/boltdb/bolt)。

containerd 进程对于metadata 和snapshotter 的boltdb 存储格式如下：

默认 db 文件 /var/lib/containerd/io.containerd.metadata.v1.bolt/meta.db

metadata格式如下 （部分数据已经省略），第一层的bucket 为v1 表明了版本好，第二层的bucket 为namespace，这里显示的default。

```text
├── v1
│   ├── default
│   │   ├── content
│   │   │   ├── blob
│   │   │   │   ├── sha256:5e8e0509e829bb8f990249135a36e81a3ecbe94294e7a185cc14616e5fad96bd
│   │   │   │   │   └── createdat=**
│   │   │   │   │   ├── labels
│   │   │   │   │   │   └── containerd.io/gc.ref.content.0=sha256:e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda
│   │   │   │   │   │   └── containerd.io/gc.ref.content.1=sha256:8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185
│   │   │   │   │   └── size=**
│   │   │   │   │   └── updatedat=**
│   │   │   │   ├── sha256:8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185
│   │   │   │   │   └── createdat=**
│   │   │   │   │   ├── labels
│   │   │   │   │   │   └── containerd.io/uncompressed=sha256:f9d9e4e6e2f0689cd752390e14ade48b0ec6f2a488a05af5ab2f9ccaf54c299d
│   │   │   │   │   └── size=**
│   │   │   │   │   └── updatedat=**
│   │   │   │   ├── sha256:cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd
│   │   │   │   │   └── createdat=**
│   │   │   │   │   ├── labels
│   │   │   │   │   │   └── containerd.io/gc.ref.content.0=sha256:5e8e0509e829bb8f990249135a36e81a3ecbe94294e7a185cc14616e5fad96bd
│   │   │   │   │   │   └── containerd.io/gc.ref.content.1=sha256:043b9724afca58fca70b1a95dfb5c88babad79c5e81242f8d42a05882a24158e
│   │   │   │   │   └── size=**
│   │   │   │   │   └── updatedat=**
│   │   │   │   ├── sha256:e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda
│   │   │   │   │   └── createdat=**
│   │   │   │   │   ├── labels
│   │   │   │   │   │   └── containerd.io/gc.ref.snapshot.overlayfs=sha256:f9d9e4e6e2f0689cd752390e14ade48b0ec6f2a488a05af5ab2f9ccaf54c299d
│   │   │   │   │   └── size=**
│   │   │   │   │   └── updatedat=**
│   │   │   ├── ingests
│   │   ├── images
│   │   │   ├── docker.io/library/busybox:latest
│   │   │   │   └── createdat=**
│   │   │   │   ├── target
│   │   │   │   │   └── digest=sha256:cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd
│   │   │   │   │   └── mediatype=application/vnd.docker.distribution.manifest.list.v2+json
│   │   │   │   │   └── size=**
│   │   │   │   └── updatedat=**
│   │   ├── leases
│   │   ├── snapshots
│   │   │   ├── overlayfs
│   │   │   │   ├── sha256:f9d9e4e6e2f0689cd752390e14ade48b0ec6f2a488a05af5ab2f9ccaf54c299d
│   │   │   │   │   └── createdat=**
│   │   │   │   │   └── name=default/4/sha256:f9d9e4e6e2f0689cd752390e14ade48b0ec6f2a488a05af5ab2f9ccaf54c299d
│   │   │   │   │   └── updatedat=**
```

snapshotter 以overlayfs 为例 默认db 文件 存放在 /var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/metadata.db

格式如下（部署数据 已经省略），第一层的bucket 为v1 代表了版本号，第二层的bucket分别为parents和snapshots，其中parents bucket 里存放的key 因为是不可读的，暂时以�显示。

```text
├── v1
│   ├── parents
│   │   └── �=k8s.io/293/ce16cfaffb0352bd2d80e0f164dffe846e71ea8b8143361bd34d286b7703977d
│   │   └── �=k8s.io/425/8f4e4854084be4341128cd311eb4320f931ec9fb154272d74c40ea05df1315d7
│   │   └── �=k8s.io/429/7416c3c71403c8c2ac96f713301796269d69e10961161dd41f5b308954ff8764
│   │   └── �=k8s.io/324/7386e266c224f05cb0795e340e960398fd359fc9d9296cad54e2e53b4ca454d7
│   │   └── =k8s.io/21/sha256:bb9294c84e1a9d926b831deb02faac58950650d0ff57dd1405eefceb25ee90d7
│   ├── snapshots
│   │   ├── default/4/sha256:f9d9e4e6e2f0689cd752390e14ade48b0ec6f2a488a05af5ab2f9ccaf54c299d
│   │   │   └── createdat=**
│   │   │   └── id=**
│   │   │   └── inodes=**
│   │   │   └── kind=
│   │   │   └── size=**
│   │   │   └── updatedat=**
│   │   ├── k8s.io/19/sha256:5bef08742407efd622d243692b79ba0055383bbce12900324f75e56f589aedb0
│   │   │   └── createdat=**
│   │   │   └── id=**
│   │   │   └── inodes=**
│   │   │   └── kind=
│   │   │   └── size=**
│   │   │   └── updatedat=**
```

### runtime-spec

根据runtime-spec 配置文件以及OCI Bundle ，我们可以使用runc 手动创建容器，具体如何使用runc创建容器，请参考官方文档 [https://github.com/opencontainers/runc](https://github.com/opencontainers/runc)。

我们在这里只需要大体知道spec 文件大体具有哪些参数（下面的实例并不是一个完整的spec文件，部分参数已省略）

```python
{
  "ociVersion": "1.0.1-dev",
  "process": {
    // 注释: process 启动参数
    "args": [
      "sleep",
      "7200000"
    ],
    // 注释: 环境变量
    "env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "KUBERNETES_PORT_443_TCP_PORT=443",
    ],
    "cwd": "/"
  },
  // 注释: root 定义了container的文件系统
  "root": {
    // 注释: 定义了文件系统的路径
    "path": "rootfs"
  },
  // 注释：mounts定义了除了root外额外的挂载点
  "mounts": [
    {
      "destination": "/dev",
      "type": "tmpfs",
      "source": "tmpfs",
      "options": [
        "nosuid",
        "strictatime",
        "mode=755",
        "size=65536k"
      ]
    }
  ],
  "linux": {
    // 注释： 通过配置resources来配置cgroup 对container进行资源限制
    "resources": {
      "memory": {
        "limit": 0
      },
      "cpu": {
        "shares": 2,
        "quota": 0,
        "period": 0
      }
    },
    "cgroupsPath": "/kubepods/besteffort/pod514d8e11-b1bd-11e8-a5f1-42010a8c0002/308096d7fcf226992edc90a74b02b28c026a5db4ca57641d1332b0eef8b9f724",
    // 注释：配置container 的namespace，namespace 支持pid,network,mount,ipc,uts,user,cgroup类型
    "namespaces": [
      {
        "type": "network",
        "path": "/proc/4904/ns/net"
      }
    ],
    "rootfsPropagation": "rslave"
  }
}
```

关于runtime-spec 的具体说明请查看[https://github.com/opencontainers/runtime-spec](https://github.com/opencontainers/runtime-spec)，有兴趣的同学可以深入了解。

### image-spec

image-spec 定义了容器镜像的格式 \(OCI Image Format\)。原话如下:

{% hint style="info" %}
The OCI Image Format partner project is the [OCI Runtime Spec project](https://github.com/opencontainers/runtime-spec). The Runtime Specification outlines how to run a "[filesystem bundle](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md)" that is unpacked on disk. At a high-level an OCI implementation would download an OCI Image then unpack that image into an OCI Runtime filesystem bundle. At this point the OCI Runtime Bundle would be run by an OCI Runtime.
{% endhint %}

下面是一个image 镜像的关系图：

关系图

![](../../../.gitbook/assets/image%20%2822%29.png)

* Image Index和Manifest的关系是"1..\*"，一个Image Index 对应多个Manifest，Image Index 是最上层Manifest 文件的索引，包含了哪些平台下的Manifest，打开一个Image Index 文件里面会有一个Manifest 列表。
* Image Manifest和Config的关系是"1..1"，一个Image Manifest 文件 对应一个Config 文件，在Image Manifest文件中有一个config 字段代表指向哪个Config 文件。
* Image Manifest和Filesystem Layers是一对多的关系，一个Image Manifest文件对应多个Filesystem Layer，在Image Manifest文件中会保存一个Layers的列表。

在containerd 下，默认这些文件都会存放在

```text
/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256
```

目录下，我们以 busybox 为例来解析镜像格式

```text
# ctr 命令下载busybox 镜像
ctr image pull docker.io/library/busybox:latest
```

以下为输出内容

```text
docker.io/library/busybox:latest:                                                 resolved       |++++++++++++++++++++++++++++++++++++++|
index-sha256:cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd:    done           |++++++++++++++++++++++++++++++++++++++|
manifest-sha256:5e8e0509e829bb8f990249135a36e81a3ecbe94294e7a185cc14616e5fad96bd: done           |++++++++++++++++++++++++++++++++++++++|
layer-sha256:8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185:    done           |++++++++++++++++++++++++++++++++++++++|
config-sha256:e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda:   done           |++++++++++++++++++++++++++++++++++++++|
elapsed: 2.0 s                                                                    total:   0.0 B (0.0 B/s)
unpacking linux/amd64 sha256:cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd...
done
```

可以看到ctr 会下载三种类型的文件分别是: index 文件：cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd, manifest 文件：5e8e0509e829bb8f990249135a36e81a3ecbe94294e7a185cc14616e5fad96bd, 一个layer 文件：8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185，一个config 文件：e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda，下面我们依次解析这几个文件（所有下载的这些文件都会默认存放到/var/lib/containerd/io.containerd.content.v1.content/blobs/sha256目录下）

**Image Index 文件:**

打开cb63aa0641a885f54de20f61d152187419e8f6b159ed11a251a09d115fdff9bd这个文件，我们会看到如下内容（部分内容已经省略）

```text
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.list.v2+json",
   "manifests": [
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 527,
         "digest": "sha256:5e8e0509e829bb8f990249135a36e81a3ecbe94294e7a185cc14616e5fad96bd",
         "platform": {
            "architecture": "amd64",
            "os": "linux"
         }
      },
      {
         "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
         "size": 527,
         "digest": "sha256:043b9724afca58fca70b1a95dfb5c88babad79c5e81242f8d42a05882a24158e",
         "platform": {
            "architecture": "arm",
            "os": "linux",
            "variant": "v5"
         }
      },
      ... 部分数据已经省略
   ]
}
```

在上述文件见的manifests 列表中，我们可以找到 amd64、linux 平台下Manifest文件

**Manifest 文件：**

打开Manifest 文件 5e8e0509e829bb8f990249135a36e81a3ecbe94294e7a185cc14616e5fad96bd ，我们会看到文件中config配置 和 layers 列表:

```text
{
   "schemaVersion": 2,
   "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
   "config": {
      "mediaType": "application/vnd.docker.container.image.v1+json",
      "size": 1497,
      "digest": "sha256:e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda"
   },
   "layers": [
      {
         "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
         "size": 733241,
         "digest": "sha256:8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185"
      }
   ]
}
```

**Config 文件:**

打开Config 文件 e1ddd7948a1c31709a23cc5b7dfe96e55fc364f90e1cebcde0773a1b5a30dcda，会看到如下内容\(部分数据已经省略\)：

```text
{
  "architecture": "amd64",
  "config": {
    "Hostname": "",
    ... 部分数据已经省略
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "sh"
    ],
    ... 部分数据已经省略
  },
  "container": "65b61a628c70e824d51fb7ac3f2ae5432b6fc52f812d0366d67df49a5e129571",
  "container_config": {
    ... 部分数据已经省略
    "Env": [
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
    ],
    "Cmd": [
      "/bin/sh",
      "-c",
      "#(nop) ",
      "CMD [\"sh\"]"
    ],
    ... 部分数据已经省略
  },
  ... 部分数据已经省略
  "history": [
    {
      "created": "2018-07-31T22:20:07.361628468Z",
      "created_by": "/bin/sh -c #(nop) ADD file:96fda64a6b725d4df5249c12e32245e2f02469ff637c38077740f4984cd883dd in / "
    },
    ... 部分数据已经省略
  ],
  "os": "linux",
  "rootfs": {
    "type": "layers",
    "diff_ids": [
      "sha256:f9d9e4e6e2f0689cd752390e14ade48b0ec6f2a488a05af5ab2f9ccaf54c299d"
    ]
  }
}
```

**Layer 文件：**

layer 文件 8c5a7da1afbc602695fcb2cd6445743cec5ff32053ea589ea9bd8773b7068185 是镜像的分层文件， 有兴趣的同学可以将此文件用tar 命令解压，查了具体内容。关于layer 的制作标准链接[:https://github.com/opencontainers/image-spec/blob/master/image-layout.md](https://github.com/opencontainers/image-spec/blob/master/layer.md#applying)

关于image-spec 的具体说明请查看 [https://github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec)，有兴趣的同学可以深入了解。

我们在大体了解image-spec 的标准后，会对后续镜像的pull和export 有很大的帮助，Containerd代码中Image 、Container 等的结构体都是按照这些标准去定义的。

### 配置文件

containerd 进程配置默认放在/etc/containerd/config.toml 文件中, 里面指定了相关配置信息（部分数据已经省略），具体含义见注释。

```text
# 注释: root dir
root = "/var/lib/containerd"
# 注释: root dir
state = "/run/containerd"
[grpc]
  address = "/run/containerd/containerd.sock"
[debug]
  level = "debug"
[plugins]
  [plugins.cgroups]
    no_prometheus = false
  [plugins.cri]
    stream_server_address = ""
    stream_server_port = "10010"
    enable_selinux = false
    # 注释: kubernetes pause 容器使用的镜像
    sandbox_image = "k8s.gcr.io/pause:3.1"
    stats_collect_period = 10
    systemd_cgroup = false
    enable_tls_streaming = false
    [plugins.cri.containerd]
      # 注释: 默认使用overlayfs 存储驱动
      snapshotter = "overlayfs"
      [plugins.cri.containerd.default_runtime]
        runtime_type = "io.containerd.runtime.v1.linux"
        # 注释: runtime 使用runc
        runtime_engine = "/usr/local/bin/runc"
        runtime_root = ""
      [plugins.cri.containerd.untrusted_workload_runtime]
        runtime_type = "io.containerd.runtime.v1.linux"
        runtime_engine = "/usr/local/bin/runsc"
        runtime_root = "/run/containerd/runsc"
    [plugins.cri.cni]
      # 注释: cni 的配置
      bin_dir = "/opt/cni/bin"
      conf_dir = "/etc/cni/net.d"
      conf_template = ""
    [plugins.cri.registry]
      [plugins.cri.registry.mirrors]
        # 注释: 配置的镜像仓库地址，可以配置多个，支持https 和http
        [plugins.cri.registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins.cri.registry.mirrors."10.146.0.2"]
          endpoint = ["http://10.146.0.2"]
```

### **Plugin**

containerd 进程里除了上文提到的GRPC plugin、CRI Plugin外，还提供了其他的服务

| Plugin | 说明 |
| :--- | :--- |
| Content Plugin |  |
| Metadata Plugin |  |
| GC Plugin |  |
| Diff Plugin |  |
| TaskMonitor Plugin |  |
| Snapshot Plugin |  |
| GRPC Plugin |  |
| Service Plugin |  |
| Runtime Plugin |  |

\*\*\*\*

### **Root/state dir**

Containerd进程的默认Root Dir 是 /var/lib/containerd目录，用于为containerd存储任何类型的持久数据。快照、内容、容器和镜像的元数据以及任何插件数据都将保存在此位置。根目录也是包含插件的命名空间。每个插件都有自己的存储数据的目录。containerd本身实际上没有任何需要存储的持久数据，它的功能来自于加载的插件。子目录列表如下

```text
/var/lib/containerd/
├── io.containerd.content.v1.content
│   ├── blobs
│   └── ingest
├── io.containerd.metadata.v1.bolt
│   └── meta.db
├── io.containerd.runtime.v1.linux
│   ├── default
│   └── example
├── io.containerd.snapshotter.v1.btrfs
└── io.containerd.snapshotter.v1.overlayfs
    ├── metadata.db
    └── snapshots
```

* **io.containerd.content.v1.content** 目录是用来存放镜像文件
* **io.containerd.metadata.v1.bolt** 目录用来存放bolddb metadata数据库文件，像namespace，container等的信息都存放到这个数据库中
* **io.containerd.snapshotter.v1.\***  是根据不同存储驱动生成的目录，我们可以了解containerd 默认使用的snapshotter: overlayfs 的 目录io.containerd.snapshotter.v1.overlayfs， 在此目录下存放了snapshotter 的boltdb文件，具体数据格式上文已经提到过。
* **io.containerd.runtime.v1.linux**  存放的是runtime 数据，子目录是根据namespace来区分，比如k8s.io 和default 目录

Containerd 进程的默认State Dir 是 /run/containerd 目录，用于存储任何类型的临时数据。套接字、pid、运行时状态、挂载点等存储在此位置。子目录列表如下

```text
/run/containerd
├── containerd.sock
├── debug.sock
├── io.containerd.runtime.v1.linux
│   └── default
│       └── redis
│           ├── config.json
│           ├── init.pid
│           ├── log.json
│           └── rootfs
│               ├── bin
│               ├── data
│               ├── dev
│               ├── etc
│               ├── home
│               ├── lib
│               ├── media
│               ├── mnt
│               ├── proc
│               ├── root
│               ├── run
│               ├── sbin
│               ├── srv
│               ├── sys
│               ├── tmp
│               ├── usr
│               └── var
└── runc
    └── default
        └── redis
            └── state.json
```

