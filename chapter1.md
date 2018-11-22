NSQ启动

apps/nsqd/nsqd.go

nsq使用了svc框架来启动一个service, Run 时, 分别调用prg 实现的 Init 和 Start 方法 启动’program’,然后监听 后两个参数的信号量, 当信号量到达, 调用 prg 实现的 Stop 方法来退出

```go
func main() {
    prg := &program{}
    if err := svc.Run(prg, syscall.SIGINT, syscall.SIGTERM); err != nil {
        log.Fatal(err)
    }
}
```

svc框架启动，相当于c语言中的deamon进程，在后台一直运行，直到接收到指定的信号

如果指定信号为空，默认是 syscall.SIGINT and syscall.SIGTERM

```go
func Run(service Service, sig ...os.Signal) error {
	env := environment{}

	//使用Init 初始化
	if err := service.Init(env); err != nil {
		return err
	}

	//调用Start，使程序持久化运行、和下面的信号处理是并行的
	if err := service.Start(); err != nil {
		return err
	}

	//信号量处理
	if len(sig) == 0 {
		sig = []os.Signal{syscall.SIGINT, syscall.SIGTERM}
	}

	signalChan := make(chan os.Signal, 1)
	signalNotify(signalChan, sig...)

	//接收信号量会阻塞在这个地方，直到系统信号量到达
	<-signalChan

	// 当信号来到, 调用stop 方法优雅的结束程序
	return service.Stop()
}
```



