# deep3

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
	i, err := ioCreate(c.id)
	if err != nil {
		return nil, err
	}
	defer func() {
		if err != nil && i != nil {
			i.Cancel()
			i.Close()
		}
	}()
	cfg := i.Config()
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
	if r.SnapshotKey != "" {
		if r.Snapshotter == "" {
			return nil, errors.Wrapf(errdefs.ErrInvalidArgument, "unable to resolve rootfs mounts without snapshotter on container")
		}

		// get the rootfs from the snapshotter and add it to the request
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
	var info TaskInfo
	for _, o := range opts {
		if err := o(ctx, c.client, &info); err != nil {
			return nil, err
		}
	}
	if info.RootFS != nil {
		for _, m := range info.RootFS {
			request.Rootfs = append(request.Rootfs, &types.Mount{
				Type:    m.Type,
				Source:  m.Source,
				Options: m.Options,
			})
		}
	}
	if info.Options != nil {
		any, err := typeurl.MarshalAny(info.Options)
		if err != nil {
			return nil, err
		}
		request.Options = any
	}
	t := &task{
		client: c.client,
		io:     i,
		id:     c.id,
	}
	if info.Checkpoint != nil {
		request.Checkpoint = info.Checkpoint
	}
	response, err := c.client.TaskService().Create(ctx, request)
	if err != nil {
		return nil, errdefs.FromGRPC(err)
	}
	t.pid = response.Pid
	return t, nil
}
```





### images service 代码分析

### cri service 代码分析

## containerd-shim 原理详解

