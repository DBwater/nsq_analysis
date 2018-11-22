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





