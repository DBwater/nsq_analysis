NSQ的启动在nsq/nsqd/nsqd.go

```go
func (n *NSQD) Main() {
    var err error
    ctx := &context{n}
    //监听tcp的连接
    n.tcpListener, err = net.Listen("tcp", n.getOpts().TCPAddress)
    if err != nil {
        n.logf(LOG_FATAL, "listen (%s) failed - %s", n.getOpts().TCPAddress, err)
        os.Exit(1)
    }
    //监听http连接
    n.httpListener, err = net.Listen("tcp", n.getOpts().HTTPAddress)
    if err != nil {
        n.logf(LOG_FATAL, "listen (%s) failed - %s", n.getOpts().HTTPAddress, err)
        os.Exit(1)
    }
    //监听https连接
    if n.tlsConfig != nil && n.getOpts().HTTPSAddress != "" {
        n.httpsListener, err = tls.Listen("tcp", n.getOpts().HTTPSAddress, n.tlsConfig)
        if err != nil {
            n.logf(LOG_FATAL, "listen (%s) failed - %s", n.getOpts().HTTPSAddress, err)
            os.Exit(1)
        }
    }

    tcpServer := &tcpServer{ctx: ctx}
    //protocol用来监听连接，并对连接进行处理
    // 封装的waitGroup，内部使用goroutine启动该服务，使用waitGroup守护改协程直到退出
    n.waitGroup.Wrap(func() {
        protocol.TCPServer(n.tcpListener, tcpServer, n.logf)
    })
    httpServer := newHTTPServer(ctx, false, n.getOpts().TLSRequired == TLSRequired)
    n.waitGroup.Wrap(func() {
        http_api.Serve(n.httpListener, httpServer, "HTTP", n.logf)
    })
    if n.tlsConfig != nil && n.getOpts().HTTPSAddress != "" {
        httpsServer := newHTTPServer(ctx, true, true)
        n.waitGroup.Wrap(func() {
            http_api.Serve(n.httpsListener, httpsServer, "HTTPS", n.logf)
        })
    }
    //以守护协程的方式启动
    n.waitGroup.Wrap(n.queueScanLoop)
    n.waitGroup.Wrap(n.lookupLoop)
    if n.getOpts().StatsdAddress != "" {
        n.waitGroup.Wrap(n.statsdLoop)
    }
}
```

在nsq/internal/protocol中有一个TCP的接口，用于处理所有的TCP连接，当 listener.Accept\(\) 接到client 的连接获取到connect 时, 交给 TCPHandler.Handle\(net.Conn\) 函数处理，不同的服务内容, 只需要实现不同的TCPHandler即可

```go
type TCPHandler interface {
	Handle(net.Conn)
}

//监听到来的连接，用tcpServer.handler进行处理
func TCPServer(listener net.Listener, handler TCPHandler, logf lg.AppLogFunc) {
	logf(lg.INFO, "TCP: listening on %s", listener.Addr())

	for {
		clientConn, err := listener.Accept()
		if err != nil {
			if nerr, ok := err.(net.Error); ok && nerr.Temporary() {
				logf(lg.WARN, "temporary Accept() failure - %s", err)
				runtime.Gosched()
				continue
			}
			// theres no direct way to detect this error because it is not exposed
			if !strings.Contains(err.Error(), "use of closed network connection") {
				logf(lg.ERROR, "listener.Accept() - %s", err)
			}
			break
		}
		go handler.Handle(clientConn)
	}

	logf(lg.INFO, "TCP: closing %s", listener.Addr())
}
```



