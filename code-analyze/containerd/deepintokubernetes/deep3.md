# 理解3

### task service 代码分析

task 是真正跟container 启动有关的服务，我们还是以ctr 为入口 以start task 为例

```bash
ctr task start --help
NAME:
   ctr tasks start - start a container that have been created

USAGE:
   ctr tasks start [command options] CONTAINER

OPTIONS:
   --null-io         send all IO to /dev/null
   --fifo-dir value  directory used for storing IO FIFOs
   --pid-file value  file path to write the task's pid
   --detach, -d      detach from the task after it has started execution
```

 代码源文件cmd/ctr/commands/tasks/start.go

```go
var startCommand = cli.Command{
	... 部分代码已经省略
	Action: func(context *cli.Context) error {
		... 部分代码已经省略
		// 注释: 创建grpc client
		client, ctx, cancel, err := commands.NewClient(context)
		... 部分代码已经省略
		// 注释：获得container的spec 数据
		spec, err := container.Spec(ctx)
		if err != nil {
			return err
		}
		... 部分代码已经省略
		// 注释 核心逻辑： 创建task
		task, err := NewTask(ctx, client, container, "", tty, context.Bool("null-io"), ioOpts, opts...)
		if err != nil {
			return err
		}
		... 部分代码已经省略
}
```

上述代码的核心是创建grpc client，获得container 的Spec 数据，调用NewTask 方法 ，源代码文件： cmd/ctr/commands/tasks/tasks\_unix.go

```go
// NewTask creates a new task
func NewTask(ctx gocontext.Context, client *containerd.Client, container containerd.Container, checkpoint string, tty, nullIO bool, ioOpts []cio.Opt, opts ...containerd.NewTaskOpts) (containerd.Task, error) {
	stdio := cio.NewCreator(append([]cio.Opt{cio.WithStdio}, ioOpts...)...)
	if checkpoint == "" {
		ioCreator := stdio
		if tty {
			ioCreator = cio.NewCreator(append([]cio.Opt{cio.WithStdio, cio.WithTerminal}, ioOpts...)...)
		}
		if nullIO {
			if tty {
				return nil, errors.New("tty and null-io cannot be used together")
			}
			ioCreator = cio.NullIO
		}
		return container.NewTask(ctx, ioCreator, opts...)
	}
	im, err := client.GetImage(ctx, checkpoint)
	if err != nil {
		return nil, err
	}
	opts = append(opts, containerd.WithTaskCheckpoint(im))
	return container.NewTask(ctx, stdio, opts...)
}
```

进入到container.NewTask 方法，源文件 container.go

```go
func (c *container) NewTask(ctx context.Context, ioCreate cio.Creator, opts ...NewTaskOpts) (_ Task, err error) {
	... 部分代码已经省略
	cfg := i.Config()
	// 注释： 创建CreateTaskRequest 结构体对象
	request := &tasks.CreateTaskRequest{
		ContainerID: c.id,
		Terminal:    cfg.Terminal,
		Stdin:       cfg.Stdin,
		Stdout:      cfg.Stdout,
		Stderr:      cfg.Stderr,
	}
	r, err := c.get(ctx)
	if err != nil {
		return nil, err
	}
	// 注释：一般情况下 SnapshotKey 为 container 的name， 可以看ctr c create 的逻辑， 在WithNewSnapshot方法
	if r.SnapshotKey != "" {
		if r.Snapshotter == "" {
			return nil, errors.Wrapf(errdefs.ErrInvalidArgument, "unable to resolve rootfs mounts without snapshotter on container")
		}

		// get the rootfs from the snapshotter and add it to the request
		// 注释： 以overlay为例 获得overlay 的挂载点数据，以overlay为例源文件 snapshots/overlay/overlay.go
		mounts, err := c.client.SnapshotService(r.Snapshotter).Mounts(ctx, r.SnapshotKey)
		if err != nil {
			return nil, err
		}
		for _, m := range mounts {
			request.Rootfs = append(request.Rootfs, &types.Mount{
				Type:    m.Type,
				Source:  m.Source,
				Options: m.Options,
			})
		}
	}
	... 部分代码已经省略
	t := &task{
		client: c.client,
		io:     i,
		id:     c.id,
	}
	... 部分代码已经省略
	// 注释:调用grpc 服务，创建task 转到源文件 services/tasks/service.go Create 方法
	response, err := c.client.TaskService().Create(ctx, request)
	if err != nil {
		return nil, errdefs.FromGRPC(err)
	}
	t.pid = response.Pid
	return t, nil
}
```

上述代码的核心逻辑为创建一个CreateTaskRequest 对象，以overlay为例，获得overlay的挂载点数据，最后调用client 的TaskService\(\).Create\(\)方法调用Containerd进程的GRPC方法。Containerd 服务端Create方法的实现在services/tasks/service.go 源文件中，稍后我们再追踪，我们这里先分析一下获得挂载点数据 的Mounts 方法

```text
c.client.SnapshotService(r.Snapshotter).Mounts(ctx, r.SnapshotKey)
```

 以overlay为例我们可以追踪到Mounts方法，是在snapshots/overlay/overlay.go 源文件实现的。

```go
// Mounts returns the mounts for the transaction identified by key. Can be
// called on an read-write or readonly transaction.
//
// This can be used to recover mounts after calling View or Prepare.
func (o *snapshotter) Mounts(ctx context.Context, key string) ([]mount.Mount, error) {
	// 注释: 以overlayfs 为例 这一步是获得 overlayfs 的db，注意，跟之前的metadata db 不一样，是另一个db 文件
	ctx, t, err := o.ms.TransactionContext(ctx, false)
	if err != nil {
		return nil, err
	}
	s, err := storage.GetSnapshot(ctx, key)
	t.Rollback()
	if err != nil {
		return nil, errors.Wrap(err, "failed to get active mount")
	}
	// 注释: 获得overlayfs 的挂载点列表
	return o.mounts(s), nil
}
```

接着我们分析Containerd 服务端Create方法，在 源文件services/tasks/service.go

```go
func (s *service) Create(ctx context.Context, r *api.CreateTaskRequest) (*api.CreateTaskResponse, error) {
	return s.local.Create(ctx, r)
}
```

转到源文件 services/tasks/local.go 下

```go
func (l *local) Create(ctx context.Context, r *api.CreateTaskRequest, _ ...grpc.CallOption) (*api.CreateTaskResponse, error) {
	... 部分代码已经省略

	// 注释：读取boltdb 数据 获得container数据
	container, err := l.getContainer(ctx, r.ContainerID)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	// 注释： 创建CreateOpts 选项，可以看到里面有Spec 数据
	opts := runtime.CreateOpts{
		Spec: container.Spec,
		IO: runtime.IO{
			Stdin:    r.Stdin,
			Stdout:   r.Stdout,
			Stderr:   r.Stderr,
			Terminal: r.Terminal,
		},
		Checkpoint: checkpointPath,
		Options:    r.Options,
	}
	// 注释: 设置挂载点的数据
	for _, m := range r.Rootfs {
		opts.Rootfs = append(opts.Rootfs, mount.Mount{
			Type:    m.Type,
			Source:  m.Source,
			Options: m.Options,
		})
	}
	// 注释；获得runtime 以linux 为例，具体实现在 linux/runtime.go文件下
	runtime, err := l.getRuntime(container.Runtime.Name)
	if err != nil {
		return nil, err
	}
	// 注释: 调用runtime 的Create 方法，以linux 为例 具体实现在linux/runtime.go 
	c, err := runtime.Create(ctx, r.ContainerID, opts)
	if err != nil {
		return nil, errdefs.ToGRPC(err)
	}
	... 部分代码已经省略
}

```

Create 方法最终调用的是runtime的Create方法，runtime 以linux为例，具体实现在linux/runtime.go 下

```go
// Create a new task
func (r *Runtime) Create(ctx context.Context, id string, opts runtime.CreateOpts) (_ runtime.Task, err error) {
	// 注释：获得namespace
	namespace, err := namespaces.NamespaceRequired(ctx)
	if err != nil {
		return nil, err
	}
    ... 部分代码已经省略
	ropts, err := r.getRuncOptions(ctx, id)
	if err != nil {
		return nil, err
	}

	// 注释：获得bundle 数据,创建对应的目录
	// 注释： workdir /var/lib/containerd/io.containerd.runtime.v1.linux/k8s.io/{id}
	// 注释; statedir /run/containerd/io.containerd.runtime.v1.linux/k8s.io/{id}/rootfs/
	bundle, err := newBundle(id,
		filepath.Join(r.state, namespace),
		filepath.Join(r.root, namespace),
		opts.Spec.Value)
	... 部分代码已经省略
	shimopt := ShimLocal(r.config, r.events)
	if !r.config.NoShim {
		// 注释 默认情况下containerd 配置文件 no_shim 配置为false，请查找配置文件中plugins->plugins.linux 配置
		// 注释 noshim == false 代表这位每个container 启动一个shim 服务
		var cgroup string
		if opts.Options != nil {
			v, err := typeurl.UnmarshalAny(opts.Options)
			if err != nil {
				return nil, err
			}
			cgroup = v.(*runctypes.CreateOptions).ShimCgroup
		}
		exitHandler := func() {
			log.G(ctx).WithField("id", id).Info("shim reaped")
			t, err := r.tasks.Get(ctx, id)
			if err != nil {
				// Task was never started or was already successfully deleted
				return
			}
			lc := t.(*Task)

			// Stop the monitor
			if err := r.monitor.Stop(lc); err != nil {
				log.G(ctx).WithError(err).WithFields(logrus.Fields{
					"id":        id,
					"namespace": namespace,
				}).Warn("failed to stop monitor")
			}

			log.G(ctx).WithFields(logrus.Fields{
				"id":        id,
				"namespace": namespace,
			}).Warn("cleaning up after killed shim")
			if err = r.cleanupAfterDeadShim(context.Background(), bundle, namespace, id, lc.pid); err != nil {
				log.G(ctx).WithError(err).WithFields(logrus.Fields{
					"id":        id,
					"namespace": namespace,
				}).Warn("failed to clen up after killed shim")
			}
		}
		// 注释 创建containerd-shim demo，并启动
		shimopt = ShimRemote(r.config, r.address, cgroup, exitHandler)
	}

	// 注释 创建containerd-shim client
	s, err := bundle.NewShimClient(ctx, namespace, shimopt, ropts)
	if err != nil {
		return nil, err
	}
	... 部分代码已经省略

	rt := r.config.Runtime
	if ropts != nil && ropts.Runtime != "" {
		rt = ropts.Runtime
	}
	sopts := &shim.CreateTaskRequest{
		ID:         id,
		Bundle:     bundle.path,
		Runtime:    rt,
		Stdin:      opts.IO.Stdin,
		Stdout:     opts.IO.Stdout,
		Stderr:     opts.IO.Stderr,
		Terminal:   opts.IO.Terminal,
		Checkpoint: opts.Checkpoint,
		Options:    opts.Options,
	}
	for _, m := range opts.Rootfs {
		sopts.Rootfs = append(sopts.Rootfs, &types.Mount{
			Type:    m.Type,
			Source:  m.Source,
			Options: m.Options,
		})
	}
	// 注释: 最终调用shim 的create 服务 来创建容器，而shim 会默认使用runc 来创建容器，具体实现 在 
	// 注释: linux/shim/service.go 源文件中
	cr, err := s.Create(ctx, sopts)
	if err != nil {
		return nil, errdefs.FromGRPC(err)
	}
	... 部分代码已经省略
	return t, nil
}
```

Create 方法 的处理逻辑

* 获得namespace
* 创建boundle目录，包括workdir 和statedir，最重要的是statedir，因为最终的spec文件和挂载点都是在statedir目录中
* 启动containerd-shim服务，默认配置是no\_shim=false 代表一个container 对应一个containerd-shim 服务，所以我们在ps aux \|grep containerd-shim时候会看到好多containerd-shim 进程，这些进程数与容器数目是一致的，如下

```bash
root      1467  0.0  0.1   7500  3028 ?        Sl   17:06   0:00 containerd-shim -namespace k8s.io -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/k8s.io/02a5825400055213dbd0b0484194ea6ba11266bcfebfd1cc49b4423e23380d1d -address /run/containerd/containerd.sock -containerd-binary /usr/local/bin/containerd
root      1545  0.0  0.1   7500  2976 ?        Sl   17:06   0:00 containerd-shim -namespace k8s.io -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/k8s.io/84892ac02542e47a7d4759dfae4b1db7005047ac1819a0c1260e75f57f795541 -address /run/containerd/containerd.sock -containerd-binary /usr/local/bin/containerd
root      1586  0.0  0.1   8908  3028 ?        Sl   17:06   0:00 containerd-shim -namespace k8s.io -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/k8s.io/b30871fdaa74ce2159e58e7f4c46ba4eeabc315decc399b53e0c3f5526d08210 -address /run/containerd/containerd.sock -containerd-binary /usr/local/bin/containerd
root      1634  0.0  0.1   7500  3020 ?        Sl   17:06   0:00 containerd-shim -namespace k8s.io -workdir /var/lib/containerd/io.containerd.runtime.v1.linux/k8s.io/0450370659a2493da8b1c4e2d36e3c68e2f90ef4515d1cc4a8c7507065a09f69 -address /run/containerd/containerd.sock -containerd-binary /usr/local/bin/containerd
```

* 创建containerd-shim client 端，调用client端的Create 方法，让containerd-shim 去调用runc 创建容器。

综上所述 我们基本上已经理清，grpc请求到containerd后，containerd会经过一系列的处理逻辑，最终会起一个containerd-shim的服务，这个shim服务当前版本是用ttrpc服务实现的 [https://github.com/containerd/ttrpc](https://github.com/containerd/ttrpc)。 之所以一个shim服务对应一个container 的设计目的是方式将影响缩小到单个container，方式shim死掉后，所有container都无法通信。未来社区也准备将shim 服务集成的Containerd中，作为Containerd 的一个plugin，后续我们会单独分析containerd-shim 进程。

### images service 代码分析

继续以ctr image pull 入口为例

```bash
NAME:
   ctr images pull - pull an image from a remote

USAGE:
   ctr images pull [command options] [flags] <ref>

DESCRIPTION:
   Fetch and prepare an image for use in containerd.

After pulling an image, it should be ready to use the same reference in a run
command. As part of this process, we do the following:

1. Fetch all resources into containerd.
2. Prepare the snapshot filesystem with the pulled resources.
3. Register metadata for the image.


OPTIONS:
   --skip-verify, -k       skip SSL certificate validation
   --plain-http            allow connections using plain HTTP
   --user value, -u value  user[:password] Registry user and password
   --refresh value         refresh token for authorization server
   --snapshotter value     snapshotter name. Empty value stands for the default value. (default: "overlayfs") [$CONTAINERD_SNAPSHOTTER]
   --label value           labels to attach to the image
   --platform value        Pull content from a specific platform
   --all-platforms         pull content from all platforms
```

函数入口 源文件 cmd/ctr/commands/images/pull.go

```go
var pullCommand = cli.Command{
	... 部分代码已经省略
	Action: func(context *cli.Context) error {
		var (
			ref = context.Args().First()
		)
		if ref == "" {
			return fmt.Errorf("please provide an image reference to pull")
		}
		ctx, cancel := commands.AppContext(context)
		defer cancel()
		// 注释： 核心代码 Fetch image
		img, err := content.Fetch(ref, context)
		if err != nil {
			return err
		}

		... 部分代码已经省略
		return err
	},
}
```

核心代码会调用content.Fetch方法，源文件 cmd/ctr/commands/content/fetch.go 

```go

```

 

之后Fetch 方法会调用client.Fetch 方法





### cri service 代码分析

## containerd-shim 原理详解

