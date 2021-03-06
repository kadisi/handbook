# 理解2

## Containerd 进程启动过程

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

上述代码的核心为下面两行，创建一个cli.App的结构体对象：app，之后调用Run 方法传入命令行参数，并最终将参数分析到cli.Context 对象中，启动app 的Action 方法，处理不同plugin service 注册到GRPC 服务最后启动grpc 服务：

```go
 app := command.App()
 app.Run(os.Args)
```

我们只需要重点关注command.App\(\) 方法即可，通常情况下containerd运行时候，不需要额外命令行参数传入，Run 方法核心任务是把os.Args的参数转成对应的参数，之后调用app的Action方法。

另外我们需要重点关注 cmd/containerd/main.go 源码文件同一级的多个buiiltins\*.go 的文件，都属于main package，如下图。

```bash
├── cmd
│   ├── containerd
│   │   ├── builtins.go
│   │   ├── builtins_btrfs_linux.go
│   │   ├── builtins_cri_linux.go
│   │   ├── builtins_linux.go
│   │   ├── builtins_unix.go
│   │   ├── builtins_windows.go
│   │   ├── main.go
```

以builtins.go 文件为例

```go
import (
   _ "github.com/containerd/containerd/diff/walking/plugin"
   _ "github.com/containerd/containerd/gc/scheduler"
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

 这些包里最终定义了实现对应GRPC 服务的接口的结构体，builtins.go 里仅仅调用了对应package 的init 方法。以/containerd/services/containers 包为例:在它的service.go 源文件里，我们可以找到init方法：

```go
func init() {
   plugin.Register(&plugin.Registration{
      Type: plugin.GRPCPlugin,
      ID:   "containers",
      Requires: []plugin.Type{
         plugin.ServicePlugin,
      },
      InitFn: func(ic *plugin.InitContext) (interface{}, error) {
         plugins, err := ic.GetByType(plugin.ServicePlugin)
         if err != nil {
            return nil, err
         }
         p, ok := plugins[services.ContainersService]
         if !ok {
            return nil, errors.New("containers service not found")
         }
         i, err := p.Instance()
         if err != nil {
            return nil, err
         }
         return &service{local: i.(api.ContainersClient)}, nil
      },
   })
}
```

 init\(\)方法会调用plugin.Register方法，注册GRPC 服务， 在InitFn 里最终return的service 结构体 它实现了GRPC proto文件里定义的接口。类似的github.com/containerd/containerd/services/tasks 包也是同样的结构。通过上面分析，我们基本上可以知道，containerd进程的GRPC服务源码实现大部分都存放在/containerd/services/目录下，而services目录下通常会有两个文件servers.go和local.go。 参照下图\(部分文件已经省略\)

```text
├── services
│   ├── containers
│   │   ├── local.go
│   │   └── service.go
│   ├── content
│   │   ├── service.go
│   │   └── store.go
│   ├── diff
│   │   ├── local.go
│   │   ├── service.go
│   ├── events
│   │   └── service.go
│   ├── healthcheck
│   │   └── service.go
│   ├── images
│   │   ├── local.go
│   │   └── service.go
│   ├── introspection
│   │   └── service.go
│   ├── leases
│   │   ├── local.go
│   │   └── service.go
│   ├── namespaces
│   │   ├── local.go
│   │   └── service.go
│   ├── services.go
│   ├── snapshots
│   │   ├── service.go
│   │   └── snapshotters.go
│   ├── tasks
│   │   ├── local.go
│   │   └── service.go
│   └── version
│       └── service.go
```

前者services.go 实现了对应api/services 目录下proto文件定义的接口，后者local.go 是对services.go 结构体方法的本地封装，如果对应的services逻辑比较复杂，一般会将处理逻辑转移到local.go源文件实现。详细了解GRPC 服务的同学 通过api/services 对应的proto文件 以及services.go 的具体实现能够很好的对应起来，我们后续会重点讲解containers/tasks/images 这三个services的实现。 api/services 目录结构如下:

```text
── api
│   ├── 1.0.pb.txt
│   ├── services
│   │   ├── containers
│   │   │   └── v1
│   │   │       ├── containers.pb.go
│   │   │       └── containers.proto
│   │   ├── content
│   │   │   └── v1
│   │   │       ├── content.pb.go
│   │   │       └── content.proto
│   │   ├── diff
│   │   │   └── v1
│   │   │       ├── diff.pb.go
│   │   │       └── diff.proto
│   │   ├── events
│   │   │   └── v1
│   │   │       ├── doc.go
│   │   │       ├── events.pb.go
│   │   │       └── events.proto
│   │   ├── images
│   │   │   └── v1
│   │   │       ├── docs.go
│   │   │       ├── images.pb.go
│   │   │       └── images.proto
│   │   ├── introspection
│   │   │   └── v1
│   │   │       ├── doc.go
│   │   │       ├── introspection.pb.go
│   │   │       └── introspection.proto
│   │   ├── leases
│   │   │   └── v1
│   │   │       ├── doc.go
│   │   │       ├── leases.pb.go
│   │   │       └── leases.proto
│   │   ├── namespaces
│   │   │   └── v1
│   │   │       ├── namespace.pb.go
│   │   │       └── namespace.proto
│   │   ├── snapshots
│   │   │   └── v1
│   │   │       ├── snapshots.pb.go
│   │   │       └── snapshots.proto
│   │   ├── tasks
│   │   │   └── v1
│   │   │       ├── tasks.pb.go
│   │   │       └── tasks.proto
│   │   └── version
│   │       └── v1
│   │           ├── version.pb.go
│   │           └── version.proto
```

回到command.App\(\)方法 在源代码 cmd/containerd/command/main.go 中实现\(部分代码已经省略\)

```go
// App returns a *cli.App instance.
func App() *cli.App {
    app := cli.NewApp()
    ... 部分代码已经省略
    app.Flags = []cli.Flag{
        ... 部分代码已经省略
    }
    ... 部分代码已经省略
    app.Action = func(context *cli.Context) error {
        ... 部分代码已经省略
        // 注释: 加载配置文件配置，数据写入到config 变量中
        if err := server.LoadConfig(context.GlobalString("config"), config); err != nil && !os.IsNotExist(err) {
            return err
        }
        ... 部分代码已经省略
        // 注释：构建server 对象
        server, err := server.New(ctx, config)
        if err != nil {
            return err
        }
        ... 部分代码已经省略
        // 注释：启动GRPC 服务
        serve(ctx, l, server.ServeGRPC)
        ... 部分代码已经省略
    }
    return app
}
```

App\(\)方法最核心的是Action function，它的主要逻辑为

* server.LoadConfig 加载默认配置文件（如果containerd进程没有单独指明的情况下）/etc/containerd/config.toml的配置到config变量中
* server.New\(\) 构建Server 对象， 此方法包含了创建多个plugin service 的核心逻辑
* serve\(\) 通过serve 方法启动GRPC 服务

## 关键代码分析

### server 对象的创建代码分析

server.New 方法是在源文件server/server.go 中实现，其主要逻辑是load所有注册的plugin，对每个plugin做配置，然后返回对应结构体实例，然后将这些实例注册到GRPC 服务中。

```go
// New creates and initializes a new containerd server
func New(ctx context.Context, config *Config) (*Server, error) {
    ... 部分代码已经省略
    // 注释： load plugin, 这些plugin 最终实现了各自的grpc service
    // LoadPlugins 会默认注册两个Plugin Content和Metadata pluginin， 其余plugin 的注册是在
    // main package 里 对应的builds*.go加载的
    plugins, err := LoadPlugins(config)
    if err != nil {
        return nil, err
    }
    // 注释：调用grpc package 的NewServer方法，创建一个rpc对象
    rpc := grpc.NewServer(
        grpc.MaxRecvMsgSize(config.GRPC.MaxRecvMsgSize),
        grpc.MaxSendMsgSize(config.GRPC.MaxSendMsgSize),
        grpc.UnaryInterceptor(grpc_prometheus.UnaryServerInterceptor),
        grpc.StreamInterceptor(grpc_prometheus.StreamServerInterceptor),
    )
    var (
        services []plugin.Service
        s = &Server{
            rpc:    rpc,
            events: exchange.NewExchange(),
            config: config,
        }
        // 注释：在所有plugin 中，initialized 相当于一个全局变量，使用Add 方法存放已经实例化好的Plugin对象
        initialized = plugin.NewPluginSet()
    )
    for _, p := range plugins {
        // 注释：每个plugin 都有一个id，这个id 是根据Type.ID 生成的
        id := p.URI()
        log.G(ctx).WithField("type", p.Type).Infof("loading plugin %q...", id)

        // 注释： initialized 存放于为每个Plugin创建的initContext 对象中，initContext 对象最终 被p.Init方法调用
        // 注释： p.Init方法会调用每个 Registration 对象的InitFn 方法，在后面我们看到的大部分的 plugin.Register（）方法里
        // 注释： 都会看到对应的Registration 对象的InitFn 方法的实现，InitFn 会使用 initContext的plugins 获得各种依赖的plugin
        initContext := plugin.NewContext(
            ctx,
            p,
            initialized,
            config.Root,
            config.State,
        )
        initContext.Events = s.events
        initContext.Address = config.GRPC.Address

        // load the plugin specific configuration if it is provided
        // 注释：根据containerd 进程的配置文件更新对应plugin Config 信息
        if p.Config != nil {
            pluginConfig, err := config.Decode(p.ID, p.Config)
            if err != nil {
                return nil, err
            }
            initContext.Config = pluginConfig
        }
        result := p.Init(initContext)
        if err := initialized.Add(result); err != nil {
            return nil, errors.Wrapf(err, "could not add plugin result to plugin set")
        }

        instance, err := result.Instance()
        if err != nil {
            if plugin.IsSkipPlugin(err) {
                log.G(ctx).WithField("type", p.Type).Infof("skip loading plugin %q...", id)
            } else {
                log.G(ctx).WithError(err).Warnf("failed to load plugin %s", id)
            }
            continue
        }
        // check for grpc services that should be registered with the server
        if service, ok := instance.(plugin.Service); ok {
            services = append(services, service)
        }
        s.plugins = append(s.plugins, result)
    }
    // register services after all plugins have been initialized
    // 注释 最终把每个service 注册到grpc 服务中
    for _, service := range services {
        if err := service.Register(rpc); err != nil {
            return nil, err
        }
    }
    return s, nil
}
```

New\(\)方法最开始会先判断必要参数:Root, State dir是否有值，若相应的检查通过后会创建Root,State 目录，之后调用LoadPlugins 方法load 所有plugin，在LoadPlugins方法中，我们会看到函数显示的注册了两个plugin：

```go
// load additional plugins that don't automatically register themselves
    plugin.Register(&plugin.Registration{
        Type: plugin.ContentPlugin,
        ID:   "content",
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            ic.Meta.Exports["root"] = ic.Root
            return local.NewStore(ic.Root)
        },
    })
    plugin.Register(&plugin.Registration{
        Type: plugin.MetadataPlugin,
        ID:   "bolt",
        Requires: []plugin.Type{
            plugin.ContentPlugin,
            plugin.SnapshotPlugin,
        },
         ... 部分代码已经省略
}
```

Type 是Content，ID 是content 的plugin 和 Type 是Metadata， ID是bolt 的plugin，这两个plugin基本上是其他所有plugin 的基础，很多其他的plugin 像ServicePlugin 都会使用这两个plugin 提供的最基础的方法。

content plugin 对应的目录是 /var/lib/containerd/io.containerd.content.v1.content 此plugin主要用来存放pull 下来的镜像文件，包括：Image Index, Image Manifest, Image Config, Layer files. 具体目录存放在blobs/sha256/ 子目录下。通过local.NewStore方法，我们可以追踪到 content/local/store.go 源文件下store 结构体对象，此对象提供如Info,ReadAt,Delete,Update,Walk,Status 等方法，主要提供对Image 文件的增删改查功能。

Metadata Plugin 主要是创建/var/lib/containerd/io.containerd.metadata.v1.bolt/meta.db boltdb 数据库文件，后续所有Containerd进程的元数据，比如container数据，都会存放到这个db文件中。

我们可以追踪一下plugin.Register方法具体做了什么操作，源代码文件 plugin/plugin.go

```go
// Register allows plugins to register
func Register(r *Registration) {
    ... 部分代码已经省略
    register.r = append(register.r, r)
}
```

Register 方法会将所有传进来了Registration对象传给一个全局变量register中的r字段中，这个全局变量register实际上是一个Registration指针数组，另外包含一个读写锁。

```go
var register = struct {
    sync.RWMutex
    r []*Registration
}{}
```

因此在上文提到的cmd/containerd/main package里的多个buiiltins\*.go 的文件，都会调用他们import 包的init方法，而这些init 方法全部调用了Register方法，最终将所有注册的Registration 对象全部传入到全局变量register中。

最后通过New\(\)方法中最后调用plugin.Graph 函数，来获得所有的有序的Registration列表，函数代码如下

```go
// Graph returns an ordered list of registered plugins for initialization.
// Plugins in disableList specified by id will be disabled.
func Graph(disableList []string) (ordered []*Registration) {
    register.RLock()
    defer register.RUnlock()
    // 注释: 先过滤掉 disable 的
    for _, d := range disableList {
        ... 部分代码已经省略
    }

    // 注释: 记录哪些Registration 已经添加了
    added := map[*Registration]bool{}
    for _, r := range register.r {
        // 注释：递归r的Requires ,最后获得一个ordered 排序好的slice 返回
        children(r.ID, r.Requires, added, &ordered)
        if !added[r] {
            ordered = append(ordered, r)
            added[r] = true
        }
    }
    return ordered
}
```

通过LoadPlugins 方法我们获得了所有plugin Registration对象后，再过for循环挨个处理每个Registration 对象

```go
for _, p := range plugins {
        // 注释：每个plugin 都有一个id，这个id 是根据Type.ID 生成的
        id := p.URI()
        log.G(ctx).WithField("type", p.Type).Infof("loading plugin %q...", id)

        // 注释： initialized 存放于为每个Plugin创建的initContext 对象中，initContext 对象最终 被p.Init方法调用
        // 注释： p.Init方法会调用每个 Registration 对象的InitFn 方法，在后面我们看到的大部分的 plugin.Register（）方法里
        // 注释： 都会看到对应的Registration 对象的InitFn 方法的实现，InitFn 会使用 initContext的plugins 获得各种依赖的plugin
        initContext := plugin.NewContext(
            ctx,
            p,
            initialized,
            config.Root,
            config.State,
        )
        initContext.Events = s.events
        initContext.Address = config.GRPC.Address

        // load the plugin specific configuration if it is provided
        // 注释：根据containerd 进程的配置文件更新对应plugin Config 信息
        if p.Config != nil {
            pluginConfig, err := config.Decode(p.ID, p.Config)
            if err != nil {
                return nil, err
            }
            initContext.Config = pluginConfig
        }
        result := p.Init(initContext)
        if err := initialized.Add(result); err != nil {
            return nil, errors.Wrapf(err, "could not add plugin result to plugin set")
        }

        instance, err := result.Instance()
        if err != nil {
            if plugin.IsSkipPlugin(err) {
                log.G(ctx).WithField("type", p.Type).Infof("skip loading plugin %q...", id)
            } else {
                log.G(ctx).WithError(err).Warnf("failed to load plugin %s", id)
            }
            continue
        }
        // check for grpc services that should be registered with the server
        if service, ok := instance.(plugin.Service); ok {
            services = append(services, service)
        }
        s.plugins = append(s.plugins, result)
    }
```

在for循环中，都会创建一个initConext 变量，其中Set对象 initialized对象会作为initContext的plugins字段，通过追踪pluginnewPluginset\(\)方法我们可以看到initialized 实际上是一个Set 类型的结构体

```go
type Set struct {
    ordered     []*Plugin // order of initialization
    byTypeAndID map[Type]map[string]*Plugin
}
```

byTypeAndID 保存了所有Plugin对象，用二级map来保存，第一级的Key 是Type 类型，第二级是Plugin 的ID name，Value 是Plugin对象指针。

以services/containers/service.go init\(\)方法里注册的Registration对象为例

```go
func init() {
    plugin.Register(&plugin.Registration{
        Type: plugin.GRPCPlugin,
        ID:   "containers",
        Requires: []plugin.Type{
            plugin.ServicePlugin,
        },
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            plugins, err := ic.GetByType(plugin.ServicePlugin)
            if err != nil {
                return nil, err
            }
            p, ok := plugins[services.ContainersService]
            if !ok {
                return nil, errors.New("containers service not found")
            }
            i, err := p.Instance()
            if err != nil {
                return nil, err
            }
            return &service{local: i.(api.ContainersClient)}, nil
        },
    })
}
```

最终保存在initialized 对象的byTypeAndID 字段key的格式为：\[plugin.GRPCPlugin\]\["containers"\] 。GRPCPlugin 的定义在源文件 plugin/plugin.go 下

```go
const (
    // AllPlugins declares that the plugin should be initialized after all others.
    AllPlugins Type = "*"
    // RuntimePlugin implements a runtime
    RuntimePlugin Type = "io.containerd.runtime.v1"
    // ServicePlugin implements a internal service
    ServicePlugin Type = "io.containerd.service.v1"
    // GRPCPlugin implements a grpc service
    GRPCPlugin Type = "io.containerd.grpc.v1"
    // SnapshotPlugin implements a snapshotter
    SnapshotPlugin Type = "io.containerd.snapshotter.v1"
    // TaskMonitorPlugin implements a task monitor
    TaskMonitorPlugin Type = "io.containerd.monitor.v1"
    // DiffPlugin implements a differ
    DiffPlugin Type = "io.containerd.differ.v1"
    // MetadataPlugin implements a metadata store
    MetadataPlugin Type = "io.containerd.metadata.v1"
    // ContentPlugin implements a content store
    ContentPlugin Type = "io.containerd.content.v1"
    // GCPlugin implements garbage collection policy
    GCPlugin Type = "io.containerd.gc.v1"
)
```

通过byTypeAndID 我们可以看出多个Plugin会属于同一个Type，例如GRPCPlugin

我们通过goland 的Find in Path功能：Shift+⌘+F 打开查找对话框，输入:

```text
Type: plugin.GRPCPlugin
```

 查看都有哪些plugin是GRPCPlugin类型

![](../../../.gitbook/assets/image%20%2831%29.png)

可以找到都有哪些源文件init 方法里注册了GRPCPlugin服务。byTypeAndID 主要的作用是让一个Regisration 通过Requires 找到对应Service的实例，InitContext 结构体会提供GetByType的方法，源文件 plugin/context.go

```go
// GetByType returns all plugins with the specific type.
func (i *InitContext) GetByType(t Type) (map[string]*Plugin, error) {
    p, ok := i.plugins.byTypeAndID[t]
    if !ok {
        return nil, errors.Wrapf(errdefs.ErrNotFound, "no plugins registered for %s", t)
    }
    return p, nil
}
```

我们继续返回到cmd/containerd/command/main.go New\(\) 方法里的for 循环,创建完initContext 对象后，会执行p.Init\(\)方法 源文件plugin/plugin.go

```go
func (r *Registration) Init(ic *InitContext) *Plugin {
    p, err := r.InitFn(ic)
    return &Plugin{
        Registration: r,
        Config:       ic.Config,
        Meta:         ic.Meta,
        instance:     p,
        err:          err,
    }
}
```

Init 方法会首先调用Registration 里的InitFn，InitFn 都会在对应service 文件的init 方法里调用。还是以container 为例， 源文件 services/containers/service.go

```go
func init() {
    plugin.Register(&plugin.Registration{
        Type: plugin.GRPCPlugin,
        ID:   "containers",
        Requires: []plugin.Type{
            plugin.ServicePlugin,
        },
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            plugins, err := ic.GetByType(plugin.ServicePlugin)
            if err != nil {
                return nil, err
            }
            p, ok := plugins[services.ContainersService]
            if !ok {
                return nil, errors.New("containers service not found")
            }
            i, err := p.Instance()
            if err != nil {
                return nil, err
            }
            return &service{local: i.(api.ContainersClient)}, nil
        },
    })
}
```

可以看到针对container GRPCPlugin 的Registration 对象的InitFn 方法，InitFn 方法最终return了一个service 对象

```go
type service struct {
    local api.ContainersClient
}
```

p.Init方法，在调用Registration对象的InitFn方法后，会返回对应的service 对象，然后将这个对象赋值给一个Plugin对象的instance 字段中。

在上面的for 循环中，调用完result := p.Init\(\) 方法后，会继续调用result.Instance\(\) 方法，实际上就是返回InitFn 方法return的service结构体。之后通过

```go
if service, ok := instance.(plugin.Service); ok {
            services = append(services, service)
}
```

将所有instance 实例，append 到services 列表中。

最终通过下面代码注册到GRPC 服务中去。

```go
// register services after all plugins have been initialized
    // 注释 最终把每个service 注册到grpc 服务中
    for _, service := range services {
        if err := service.Register(rpc); err != nil {
            return nil, err
        }
    }
```

service 接口对象我们以container 为例 源文件 services/containers/service.go

```go
func (s *service) Register(server *grpc.Server) error {
    api.RegisterContainersServer(server, s)
    return nil
}
```

这个service 结构体对象即是上文提到的InitFn 方法中return 的对象。 Register 最终调用的是RegisterContainersServer\(\)方法，通过追踪我们可以看到在源文件 api/services/containers/v1/containers.pb.go

```go
func RegisterContainersServer(s *grpc.Server, srv ContainersServer) {
    s.RegisterService(&_Containers_serviceDesc, srv)
}
```

这个containerd.pb.go 是通过相同目录下的containers.proto 文件来自动生成的，相信了解GRPC 服务的同学看到这一步后，就会恍然大悟了。

综上所述，Containerd GRPC 消息和服务的定义在 api目录下，api 目录分别定义了containers，tasks，images，content等proto描述文件，以及根据对应描述文件自动生成的pb.go 源码文件，源码文件定义了具体的消息结构体和Service接口。而services 目录下的各种service.go 文件里最终定义了实现Service接口的结构体。

接下来我们重点分析几个service。

### container service 代码分析

我们以 ctr container create 命令行为入口，来逐步理解container的创建过程。

源码文件 cmd/ctr/commands/containers/containers.go

```go
var createCommand = cli.Command{
    ... 部分代码已经省略
    Action: func(context *cli.Context) error {
        var (
            id  = context.Args().Get(1)
            ref = context.Args().First()
        )
        ... 部分代码已经省略
        // 注释：创建grpc client 
        client, ctx, cancel, err := commands.NewClient(context)
        ... 部分代码已经省略
        _, err = run.NewContainer(ctx, client, context)
        ... 部分代码已经省略
    },
}
```

Action里主要是创建grpc client 之后调用run.NewContainer 方法，在ctr 代码中，我们可以发现所有的Command 逻辑中，都会首先创建一个grpc client。为了加深理解，我们单独分析一下client 的创建流程， NewClient 方法在源文件 cmd/ctr/commands/client.go 下 ：

```go
// NewClient returns a new containerd client
func NewClient(context *cli.Context) (*containerd.Client, gocontext.Context, gocontext.CancelFunc, error) {
    // 注释：首先会获得命令行参数传过来的address 值，默认是 /run/containerd/containerd.sock
    // 注释: 调用containerd.New 方法创建client，client 的结构体定义在 client.go 源文件中
    client, err := containerd.New(context.GlobalString("address"))
    if err != nil {
        return nil, nil, nil, err
    }
    // 注释: 调用AppContext 创建新的context，设置超时时间和namespace
    ctx, cancel := AppContext(context)
    return client, ctx, cancel, nil
}
```

这里需要提醒的是Containerd进程支持不同namespace，在上文提到的boltdb 数据结构中我们也看到namespace的影子。containerd提供了一个完全具有namespace的API，因此多个使用者都可以使用单个containerd实例，而不会彼此冲突。namespace允许单个守护进程内提供多租户服务。例如kubernetes与containerd 集成，kubelet创建的container都会在containerd的k8s.io namespace中，普通用户通过ctr 命令默认创建的容器都会放到default namespace 中。

接下来我们看一下New方法

```go
// New returns a new containerd client that is connected to the containerd
// instance provided by address
func New(address string, opts ...ClientOpt) (*Client, error) {
    ... 部分代码已经省略
    c := &Client{
        runtime: fmt.Sprintf("%s.%s", plugin.RuntimePlugin, runtime.GOOS),
    }
    if copts.services != nil {
        c.services = *copts.services
    }
    if address != "" {
        gopts := []grpc.DialOption{
            grpc.WithBlock(),
            grpc.WithInsecure(),
            grpc.WithTimeout(60 * time.Second),
            grpc.FailOnNonTempDialError(true),
            grpc.WithBackoffMaxDelay(3 * time.Second),
            grpc.WithDialer(dialer.Dialer),

            // TODO(stevvooe): We may need to allow configuration of this on the client.
            grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(defaults.DefaultMaxRecvMsgSize)),
            grpc.WithDefaultCallOptions(grpc.MaxCallSendMsgSize(defaults.DefaultMaxSendMsgSize)),
        }
        if len(copts.dialOptions) > 0 {
            gopts = copts.dialOptions
        }
        if copts.defaultns != "" {
            unary, stream := newNSInterceptors(copts.defaultns)
            gopts = append(gopts,
                grpc.WithUnaryInterceptor(unary),
                grpc.WithStreamInterceptor(stream),
            )
        }
        connector := func() (*grpc.ClientConn, error) {
            conn, err := grpc.Dial(dialer.DialAddress(address), gopts...)
            if err != nil {
                return nil, errors.Wrapf(err, "failed to dial %q", address)
            }
            return conn, nil
        }
        conn, err := connector()
        if err != nil {
            return nil, err
        }
        c.conn, c.connector = conn, connector
    }
    if copts.services == nil && c.conn == nil {
        return nil, errors.New("no grpc connection or services is available")
    }
    return c, nil
}
```

New\(\)方法主要是通过创建多个grpc 的DialOption，然后通过grpc.Dail 方法创建一个coonection，赋值给Client 结构体变量，最后返回Client 变量。默认使用了以下选项

```go
grpc.WithBlock(),
grpc.WithInsecure(),
grpc.WithTimeout(60 * time.Second),
grpc.FailOnNonTempDialError(true),
grpc.WithBackoffMaxDelay(3 * time.Second),
grpc.WithDialer(dialer.Dialer),
// TODO(stevvooe): We may need to allow configuration of this on the client.
grpc.WithDefaultCallOptions(grpc.MaxCallRecvMsgSize(defaults.DefaultMaxRecvMsgSize)),
grpc.WithDefaultCallOptions(grpc.MaxCallSendMsgSize(defaults.DefaultMaxSendMsgSize)),
```

接下来我们接着看run.NewContainer方法 源文件 cmd/ctr/commands/containers/containers.go，在ctr 代码中我们已经简单分析了这个方法的主要流程

```go
// NewContainer creates a new container
func NewContainer(ctx gocontext.Context, client *containerd.Client, context *cli.Context) (containerd.Container, error) {
    ... 部分代码已经省略
    // oci.WithImageConfig (WithUsername, WithUserID) depends on rootfs snapshot for resolving /etc/passwd.
    // So cOpts needs to have precedence over opts.
    // TODO: WithUsername, WithUserID should additionally support non-snapshot rootfs
    return client.NewContainer(ctx, id, cOpts...)
}
```

NewContainer方法主要是生成cOpts数据 （\[\]containerd.NewContainerOpts）从变量声明我们可以指定，cOpts 的目的是通过for循环的方式给一个Containerd 对象字段赋值，具体实现在client.NewContainer方法中 源文件 client.go

```go
// NewContainer will create a new container in container with the provided id
// the id must be unique within the namespace
func (c *Client) NewContainer(ctx context.Context, id string, opts ...NewContainerOpts) (Container, error) {
    ... 部分代码已经省略
    container := containers.Container{
        ID: id,
        Runtime: containers.RuntimeInfo{
            Name: c.runtime,
        },
    }
    // 注释：通过opts 给container 变量字段赋值
    for _, o := range opts {
        if err := o(ctx, c, &container); err != nil {
            return nil, err
        }
    }
    // 注释：调用grpc 接口，ContainerService()方法 默认返回的是remoteContainers 结构体对象
    // 注释: remoteContainers 结构体 在 client.go 文件里声明
    r, err := c.ContainerService().Create(ctx, container)
    if err != nil {
        return nil, err
    }
    return containerFromRecord(c, r), nil
}
```

NewContainer 方法会默认创建一个Container的结构体变量，然后通过opts 对结构体变量进行赋值，之后调用c.ContainerService\(\).Create\(\)方法，Create方法的实现在源文件 containerstore.go

```go
func (r *remoteContainers) Create(ctx context.Context, container containers.Container) (containers.Container, error) {
    created, err := r.client.Create(ctx, &containersapi.CreateContainerRequest{
        Container: containerToProto(&container),
    })
    if err != nil {
        return containers.Container{}, errdefs.FromGRPC(err)
    }
    return containerFromProto(&created.Container), nil
}
```

而服务器端Create 的实现是在 services/containers/service.go 源文件下

```go
func (s *service) Create(ctx context.Context, req *api.CreateContainerRequest) (*api.CreateContainerResponse, error) {
    return s.local.Create(ctx, req)
}
```

之前我们在上文中也提到过，Containerd 的GRPC 服务端的实现大部分在services目录下对应资源的service.go， 可见Containerd的代码还是很规范的，和对于我们进行代码追踪也有很大的帮助。通过上述代码我们看到Create方法实际上调用的s.local.Create 方法，我们先看local的值是怎么来的， 回到services/containers/service.go 文件的init方法

```go
func init() {
    plugin.Register(&plugin.Registration{
        ... 部分代码已经省略
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            plugins, err := ic.GetByType(plugin.ServicePlugin)
            if err != nil {
                return nil, err
            }
            p, ok := plugins[services.ContainersService]
            if !ok {
                return nil, errors.New("containers service not found")
            }
            i, err := p.Instance()
            if err != nil {
                return nil, err
            }
            return &service{local: i.(api.ContainersClient)}, nil
        },
    })
}
```

local 变量实际上是p.Instance\(\), 而p 是通过plugins\[services.ContainersService\]找到的，通过goland 的find in path 功能，我们可以看到

![](../../../.gitbook/assets/image%20%2827%29.png)

这个i变量实际上是在于service.go 同一级的local.go 文件的init方法中定义的local 变量

源文件 services/containers/local.go

```go
func init() {
    plugin.Register(&plugin.Registration{
        ... 部分代码已经省略
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            m, err := ic.Get(plugin.MetadataPlugin)
            if err != nil {
                return nil, err
            }
            return &local{
                db:        m.(*metadata.DB),
                publisher: ic.Events,
            }, nil
        },
    })
}
```

我们可以看到 local 实际上主要保存了metaadta.DB 对象，这样service.go 里Create方法 调用的s.local.Create 方法就能对应上了，我们来看local.go 的Create方法

```go
func (l *local) Create(ctx context.Context, req *api.CreateContainerRequest, _ ...grpc.CallOption) (*api.CreateContainerResponse, error) {
    var resp api.CreateContainerResponse

    if err := l.withStoreUpdate(ctx, func(ctx context.Context, store containers.Store) error {
        container := containerFromProto(&req.Container)

        created, err := store.Create(ctx, container)
        if err != nil {
            return err
        }

        resp.Container = containerToProto(&created)

        return nil
    }); err != nil {
        return &resp, errdefs.ToGRPC(err)
    }
    ... 部分代码已经省略
    return &resp, nil
}
```

这段函数主要逻辑在withStoreUpdate 方法中饿store.Create 方法，通过追踪withStoreUpdate方法，可以找到store 是由containerStore 结构体实现 源文件 metadata/containers.go store.Create 方法

```go
func (s *containerStore) Create(ctx context.Context, container containers.Container) (containers.Container, error) {
	namespace, err := namespaces.NamespaceRequired(ctx)
	if err != nil {
		return containers.Container{}, err
	}

	if err := validateContainer(&container); err != nil {
		return containers.Container{}, errors.Wrap(err, "create container failed validation")
	}

	bkt, err := createContainersBucket(s.tx, namespace)
	if err != nil {
		return containers.Container{}, err
	}

	cbkt, err := bkt.CreateBucket([]byte(container.ID))
	if err != nil {
		if err == bolt.ErrBucketExists {
			err = errors.Wrapf(errdefs.ErrAlreadyExists, "container %q", container.ID)
		}
		return containers.Container{}, err
	}

	container.CreatedAt = time.Now().UTC()
	container.UpdatedAt = container.CreatedAt
	if err := writeContainer(cbkt, &container); err != nil {
		return containers.Container{}, errors.Wrapf(err, "failed to write container %q", container.ID)
	}

	return container, nil
}

```

Create 方法bucket的创建顺序依次为 namespace bucket-&gt;container bucket-&gt;container id bucket-&gt;\(createdat key ,extensions bucket, image key, labels bucket, runtime bucket, spec key ,updatedat key 等等\)。

至此container 的创建流程已经梳理完毕。通过源码我们可以了解到，在containerd进程中，container资源的创建基本上是存数据库的操作。真正跟container 实例有关的是task资源后续我们会分析。

