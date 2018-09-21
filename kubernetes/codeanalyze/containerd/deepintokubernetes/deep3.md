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





### images service 代码分析

### cri service 代码分析

## containerd-shim 原理详解

