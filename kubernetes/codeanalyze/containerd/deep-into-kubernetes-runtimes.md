---
description: 深入理解Kubernetes
---

# Deep Into kubernetes - runtimes

## 第6章 容器运行时

### 前言

随着容器技术的快速发展，生态逐渐地从单体化运行容器应用，发展为支持大规模容器编排，容器成为了进程执行单元。其中容器编排以Kubernetes,Mesos为代表。面对这些竞争，2016年6月，Docker宣布在Docker Engine中内置Swarm，极大简化了容器编排的复杂性。Google发起CRI（Container RuntimeInterface容器运行时接口）项目，通过shim的抽象层使得调度框架支持不同的容器引擎实现。

面对这些挑战，2016年12月，Docker宣布将Docker Engine的核心组件Containerd捐赠到一个新的开源社区独立发展和运营，目标是提供一个标准化的容器运行时，注重简单、 健壮性和可移植性。由于Containerd只包含容器管理最基本的能力，上层框架可以有更大的灵活性来提供容器的调度和编排能力。

所以Containerd是容器技术标准化之后的产物，为了能够兼容OCI标准，将容器运行时及其管理功能从Docker Daemon剥离。理论上，即使不运行dockerd，也能够直接通过Containerd来管理容器。（当然，Containerd本身也只是一个守护进程，容器的实际运行时由后面介绍的runC控制。）

Containerd向上为Docker Daemon提供了gRPC接口，使得Docker Daemon屏蔽下面的结构变化，确保原有接口向下兼容。向下通过Containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题。 1.10以后，Containerd 集成Cri-containerd插件，支持Kubelet CRI规范，可以让Kubelet 直接调用。

### Containerd 源码结构和编译步骤

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

![](../../../.gitbook/assets/image%20%283%29.png)

由于我们只是在Mac 下查看Containerd 的代码，并不需要在Mac下编译，最终的运行环境是Linux，所以我们只关心跟Linux 平台相关的代码， 因此需要对Goland进行设置，打开Goland的Preferences -&gt; Go -&gt; Vendoring & Build Tag。 OS 选择linux，Arch 选择Default \(amd64\) 

![](../../../.gitbook/assets/image%20%2812%29.png)

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

### Containerd 架构

Containerd是一个遵循行业标准的容器运行时，它强调简单性，健壮性和可移植性。 可以管理其主机系统的完整容器生命周期，包括：映像传输和存储，容器执行，监视，存储和网络等。

Containerd涉及之初旨在嵌入到更大的系统中，例如Kubernetes，而不是由开发人员或最终用户直接使用。因此Containerd 对于最终用户而言在使用方面并不如Docker 那么友好。不过Containerd也提供ctr 命令行供测试和调试用。

![](../../../.gitbook/assets/image%20%2813%29.png)

Containerd 最上层提供一个最主要的GRPC 接口，供Docker 或者Kubelet 去调用， 第二层是各种资源对象，其中最主要的有Content，Snapshot，Images，Containers，Task 等资源对象， 其中metadata数据会存放到boltdb 本地数据库中， 而下载的Image manifest 等文件存放到本地特定目录下，最下层是Runtimes，Containerd 通过containerd-shim 默认调用runc 来实际创建容器。

![](../../../.gitbook/assets/image%20%285%29.png)

Containerd 在1.1版本已经将Cri-containerd作为Plugin的形式对外提供服务，即Containerd 代码中的 CRI Plugin， 因此与kubelet集成时，已经不需要部署单独的Cri-Containerd 服务。CRI Plugin 实现了image service 和 runtime service 接口，当CRI Plugin 接受到kubelet CRI client 的gRPC请求后， 会创建一个client 连接自身的GRPC plugin 服务， 调用相关的container，task，和snapshots等接口。同时CRI plugin 还会调用CNI接口，来进行对Pod 网络设置。

Containerd 每次创建一个container 都会给对应的container 起一个containerd-shim 服务，Containerd-shim 对外提供rRPC 服务，社区也正准备让Containerd-shim 支持gRPC, Containerd-shim 的目的是向上对接Containerd 向下可以支持不同的OCI runtime 实现， 现在Containerd-shim 默认使用的runc，通过runc来最终创建container。每个container 分配单独的Containerd-shim 服务的好处是，防止Containerd-shim 服务挂掉后，导致Containerd 无法对其他的Container进行访问。

### ctr 进程源码分析

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

#### ctr进程启动启动过程

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

### Containerd 进程源码分析

Containerd 进程是整个Containerd服务的核心模块，往上给kubelet，ctr，ctrctl 等提供GRPC接口，往下调用Containerd-shim 进程操作每个Container。

Containerd 的其他介绍 \*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*\*

在分析Containerd 之前我们需要提前了解一下相关的知识，这样对于我们快速了解Containerd会有很好的帮助

* Containerd 进程默认的本地数据库为BoltDB以及数据在BoltDB中的存储格式 
* runtime-spec OCI 为容器指定的规范  
* image-spec OCI为Image 指定的规范 [https://github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec)

#### BoltDB

BoltDB是一个由golang实现嵌入式key/value的数据库。BoltDB提供的API来高效的存取数据。而且支持完全可序列化的ACID事务，让应用程序可以更简单的处理复杂操作。BoltDB 接口中封装了比如：更新，查看，批量更新等事务的接口，方便开发人员直接使用。

```text
func (db *DB) Update(fn func(*Tx) error) error 
func (db *DB) View(fn func(*Tx) error) error 
func (db *DB) Batch(fn func(*Tx) error) error 
```

建议在看Containerd 代码前，先对BoltDB 熟悉一遍，了解db 中bucket 的创建，嵌套查询，key/value 的更新查询等， 这让我们以后分析Containerd代码会更加顺畅。具体了解和使用请访问 [https://github.com/boltdb/bolt](https://github.com/boltdb/bolt)。

#### runtime-spec

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

#### image-spec

image-spec 定义了容器镜像的格式 \(OCI Image Format\)。 image-spec 项目是runtime-spec 的伙伴项目，后者定义了





The OCI Image Format partner project is the [OCI Runtime Spec project](https://github.com/opencontainers/runtime-spec). The Runtime Specification outlines how to run a "[filesystem bundle](https://github.com/opencontainers/runtime-spec/blob/master/bundle.md)" that is unpacked on disk. At a high-level an OCI implementation would download an OCI Image then unpack that image into an OCI Runtime filesystem bundle. At this point the OCI Runtime Bundle would be run by an OCI Runtime.



[https://github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec)



#### metadata 数据结构

#### 各个plugin 干什么的 以及注册和依赖关系





















本源码分析以linux 平台为准 containerd 使用grpc 框架，主要目录结构如下

```text
-- api
-- cmd
-- content
-- defaults
-- filters
-- images
-- leases
-- metadata
-- metrics
-- mount
-- namespaces
-- oci
-- pkg
-- services
-- snapshots
-- sys
-- vendor
  -- github.com
    -- containerd
      -- cri
-- client.go
-- container.go
-- containerstore.go
-- image.go
-- image_store.go
-- lease.go
-- namespaces.go
-- process.go
-- service.go
-- snapshot_default_linux.go
-- task.go
```

containerd 入口 cmd-containerd-main.go

```text
func main() {
  app := command.App()
  if err := app.Run(os.Args); err != nil {
    fmt.Fprintf(os.Stderr, "containerd: %s\n", err)
    os.Exit(1)
  }
}
```

plugin service 的加载 全在 buildins\*.go 文件中，以buildins.go 文件为例:

```text
import (
  _ "github.com/containerd/containerd/diff/walking/plugin"
  _ "github.com/containerd/containerd/gc/scheduler"
  _ "github.com/containerd/containerd/runtime/restart/monitor"
  _ "github.com/containerd/containerd/services/containers"
  _ "github.com/containerd/containerd/services/content"
  _ "github.com/containerd/containerd/services/diff"
  _ "github.com/containerd/containerd/services/events"
  _ "github.com/containerd/containerd/services/healthcheck"
  _ "github.com/containerd/containerd/services/images"
  _ "github.com/containerd/containerd/services/introspection"
  _ "github.com/containerd/containerd/services/leases"
  _ "github.com/containerd/containerd/services/namespaces"
  _ "github.com/containerd/containerd/services/snapshots"
  _ "github.com/containerd/containerd/services/tasks"
  _ "github.com/containerd/containerd/services/version"
)
```

### containerd-shim 原理详解





### 引用文档

[https://jimmysong.io/posts/kubernetes-open-interfaces-cri-cni-csi/](https://jimmysong.io/posts/kubernetes-open-interfaces-cri-cni-csi/)

