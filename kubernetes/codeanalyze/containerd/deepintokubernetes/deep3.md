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
		if err != nil {
			return err
		}
		defer cancel()
		container, err := client.LoadContainer(ctx, id)
		if err != nil {
			return err
		}

		spec, err := container.Spec(ctx)
		if err != nil {
			return err
		}

		var (
			tty    = spec.Process.Terminal
			opts   = getNewTaskOpts(context)
			ioOpts = []cio.Opt{cio.WithFIFODir(context.String("fifo-dir"))}
		)
		task, err := NewTask(ctx, client, container, "", tty, context.Bool("null-io"), ioOpts, opts...)
		if err != nil {
			return err
		}
		defer task.Delete(ctx)
		if context.IsSet("pid-file") {
			if err := commands.WritePidFile(context.String("pid-file"), int(task.Pid())); err != nil {
				return err
			}
		}
		statusC, err := task.Wait(ctx)
		if err != nil {
			return err
		}

		var con console.Console
		if tty {
			con = console.Current()
			defer con.Reset()
			if err := con.SetRaw(); err != nil {
				return err
			}
		}
		if err := task.Start(ctx); err != nil {
			return err
		}
		if tty {
			if err := HandleConsoleResize(ctx, task, con); err != nil {
				logrus.WithError(err).Error("console resize")
			}
		} else {
			sigc := commands.ForwardAllSignals(ctx, task)
			defer commands.StopCatch(sigc)
		}

		status := <-statusC
		code, _, err := status.Result()
		if err != nil {
			return err
		}
		if _, err := task.Delete(ctx); err != nil {
			return err
		}
		if code != 0 {
			return cli.NewExitError("", int(code))
		}
		return nil
	},
}
```





### images service 代码分析

### cri service 代码分析

## containerd-shim 原理详解

