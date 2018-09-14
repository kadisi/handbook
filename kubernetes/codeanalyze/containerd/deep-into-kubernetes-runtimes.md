# Deep Into kubernetes - runtimes

## 深入理解Kubernetes

### 第6章 容器运行时

随着容器技术的快速发展，生态逐渐地从单体化运行容器应用，发展为支持大规模容器编排，容器成为了进程执行单元。其中容器编排以Kubernetes,Mesos为代表。面对这些竞争，2016年6月，Docker宣布在Docker Engine中内置Swarm，极大简化了容器编排的复杂性。Google发起CRI（Container RuntimeInterface容器运行时接口）项目，通过shim的抽象层使得调度框架支持不同的容器引擎实现。

面对这些挑战，2016年12月，Docker宣布将Docker Engine的核心组件Containerd捐赠到一个新的开源社区独立发展和运营，目标是提供一个标准化的容器运行时，注重简单、 健壮性和可移植性。由于Containerd只包含容器管理最基本的能力，上层框架可以有更大的灵活性来提供容器的调度和编排能力。

所以Containerd是容器技术标准化之后的产物，为了能够兼容OCI标准，将容器运行时及其管理功能从Docker Daemon剥离。理论上，即使不运行dockerd，也能够直接通过Containerd来管理容器。（当然，Containerd本身也只是一个守护进程，容器的实际运行时由后面介绍的runC控制。）

Containerd向上为Docker Daemon提供了gRPC接口，使得Docker Daemon屏蔽下面的结构变化，确保原有接口向下兼容。向下通过Containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题。 1.10以后，Containerd 集成Cri-containerd插件，支持Kubelet CRI规范，可以让Kubelet 直接调用。

#### Containerd 源码结构和编译步骤

Containerd 的源码现在托管在GitHub上， 地址为[https://github.com/containerd/containerd](https://github.com/containerd/containerd)。

本章节后续所有的代码分析都假设是Linux 、go1.10.2 ，Containerd 的代码分支 release/1.1 commit 号为 57508dcb0b5776efaacd0828ed42f819fab5ba07 的环境。

 Containerd的编译命令很简单，只需要在代码的根目录下执行 make 即可，最后make命令会在bin目录下编译出:

```text
mkdir -p $GOPATH/src/github.com/containerd
cd  $GOPATH/src/github.com/containerd
git clone git@github.com:containerd/containerd.git
cd containerd
make

```

####  Cri/Containerd 原理详解





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

#### containerd-shim 原理详解





### 引用文档

[https://jimmysong.io/posts/kubernetes-open-interfaces-cri-cni-csi/](https://jimmysong.io/posts/kubernetes-open-interfaces-cri-cni-csi/)

