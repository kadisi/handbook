# Deep Into kubernetes - runtimes 2

### Containerd 进程启动过程

containerd 入口 源代码位置如下：cmd/containerd/main.go

```go
func main() {
  app := command.App()
  if err := app.Run(os.Args); err != nil {
    fmt.Fprintf(os.Stderr, "containerd: %s\n", err)
    os.Exit(1)
  }
}
```

上述代码的核心为下面两行，创建一个cli.App的结构体对象：app

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



#### 各个plugin 干什么的 以及注册和依赖关系





## containerd-shim 原理详解





### 引用文档

{% embed data="{\"url\":\"https://jimmysong.io/posts/kubernetes-open-interfaces-cri-cni-csi/\",\"type\":\"link\",\"title\":\"Kubernetes中的开放接口CRI、CNI、CSI\",\"description\":\"容器运行时接口、容器网络接口、容器存储接口解析\",\"icon\":{\"type\":\"icon\",\"url\":\"https://res.cloudinary.com/jimmysong/raw/upload/rootsongjc-hugo/favicon.ico\",\"aspectRatio\":0},\"thumbnail\":{\"type\":\"thumbnail\",\"url\":\"https://ws4.sinaimg.cn/large/006tKfTcly1ft1oje0atgj31kw1kzk1j.jpg\",\"aspectRatio\":0}}" %}



image 

{% embed data="{\"url\":\"https://segmentfault.com/a/1190000009309347\",\"type\":\"link\",\"title\":\"走进docker\(02\)：image\(镜像\)是什么？ - 个人文章 - SegmentFault 思否\",\"description\":\"上一篇介绍了hello-world的大概流程，那么hello-world的image里面到底包含了些什么呢？里面的格式是怎么样的呢？\",\"icon\":{\"type\":\"icon\",\"url\":\"https://static.segmentfault.com/v-5b973eab/global/img/touch-icon.png\",\"aspectRatio\":0},\"thumbnail\":{\"type\":\"thumbnail\",\"url\":\"https://static.segmentfault.com/v-5b973eab/global/img/touch-icon.png\",\"aspectRatio\":0}}" %}





