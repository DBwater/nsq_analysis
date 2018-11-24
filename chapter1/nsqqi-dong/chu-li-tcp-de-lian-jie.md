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

可以在nsq/nsqd/tcp.go看到nsq的tcpServer的具体实现

```go
type tcpServer struct {
    ctx *context
}

func (p *tcpServer) Handle(clientConn net.Conn) {
    p.ctx.nsqd.logf(LOG_INFO, "TCP: new client(%s)", clientConn.RemoteAddr())

    // The client should initialize itself by sending a 4 byte sequence indicating
    // the version of the protocol that it intends to communicate, this will allow us
    // to gracefully upgrade the protocol away from text/line oriented to whatever...
    //读取tcp流的前四个字节，用于选择是哪种协议
    buf := make([]byte, 4)
    _, err := io.ReadFull(clientConn, buf)
    if err != nil {
        p.ctx.nsqd.logf(LOG_ERROR, "failed to read protocol version - %s", err)
        return
    }
    protocolMagic := string(buf)

    p.ctx.nsqd.logf(LOG_INFO, "CLIENT(%s): desired protocol magic '%s'",
        clientConn.RemoteAddr(), protocolMagic)
    //tcp时候的通信协议，支持nsq的扩展性目前支持v2协议
    var prot protocol.Protocol
    switch protocolMagic {
    case "  V2":
        prot = &protocolV2{ctx: p.ctx}
    default:
        // 其他协议则发送E_BAD_PROTOCOL并关闭连接
        protocol.SendFramedResponse(clientConn, frameTypeError, []byte("E_BAD_PROTOCOL"))
        clientConn.Close()
        p.ctx.nsqd.logf(LOG_ERROR, "client(%s) bad protocol magic '%s'",
            clientConn.RemoteAddr(), protocolMagic)
        return
    }
    //通过prot的IO循环时间来处理每一个连接
    err = prot.IOLoop(clientConn)
    if err != nil {
        p.ctx.nsqd.logf(LOG_ERROR, "client(%s) - %s", clientConn.RemoteAddr(), err)
        return
    }
}
```

tcpServe主要是通过判断哪种协议，然后根据协议来处理每一个连接

在nsq/nsqd/protocol\_v2.go的IOLoop具体实现了处理的细节

```go
func (p *protocolV2) IOLoop(conn net.Conn) error {
    var err error
    var line []byte
    var zeroTime time.Time
    //原子操作防止多线程冲突,创建客户端的序列号
    clientID := atomic.AddInt64(&p.ctx.nsqd.clientIDSequence, 1)
    client := newClientV2(clientID, conn, p.ctx)

    //把需要投递给客户端的消息从chan里面抽取出来（处理订阅消息的发送）

    messagePumpStartedChan := make(chan bool)
    go p.messagePump(client, messagePumpStartedChan)
    <-messagePumpStartedChan

    //处理客户端的请求
    for {
        //如果设置了心跳则读超时时间设置为心跳的两倍
        //因为在一个心跳周期，肯定会有一个报文到达，read不会超时
        //如果没有设置心跳，则不设置读超时
        if client.HeartbeatInterval > 0 {
            client.SetReadDeadline(time.Now().Add(client.HeartbeatInterval * 2))
        } else {
            client.SetReadDeadline(zeroTime)
        }
        //读取客户端消息，以\n符号分割
        // ReadSlice does not allocate new space for the data each request
        // ie. the returned slice is only valid until the next call to it
        line, err = client.Reader.ReadSlice('\n')
        if err != nil {
            if err == io.EOF {
                err = nil
            } else {
                err = fmt.Errorf("failed to read command - %s", err)
            }
            break

        }
        //v2协议一行就是一个命令
        line = line[:len(line)-1]

        // windows版本的回车可能为\r\n，特殊处理一下
        if len(line) > 0 && line[len(line)-1] == '\r' {
            line = line[:len(line)-1]
        }

        // 每个命令, 用 " "空格来划分 参数
        params := bytes.Split(line, separatorBytes)

        p.ctx.nsqd.logf(LOG_DEBUG, "PROTOCOL(V2): [%s] %s", client, params)

        var response []byte
        //执行相应的命令
        response, err = p.Exec(client, params)
        //如果执行出错，则输出对应的log信息
        if err != nil {
            ctx := ""
            if parentErr := err.(protocol.ChildErr).Parent(); parentErr != nil {
                ctx = " - " + parentErr.Error()
            }
            p.ctx.nsqd.logf(LOG_ERROR, "[%s] - %s%s", client, err, ctx)

            sendErr := p.Send(client, frameTypeError, []byte(err.Error()))
            if sendErr != nil {
                p.ctx.nsqd.logf(LOG_ERROR, "[%s] - %s%s", client, sendErr, ctx)
                break
            }

            // errors of type FatalClientErr should forceably close the connection
            if _, ok := err.(*protocol.FatalClientErr); ok {
                break
            }
            continue
        }

        // 如果命令有需要 '响应' 的, 则发送响应
        if response != nil {
            err = p.Send(client, frameTypeResponse, response)
            if err != nil {
                err = fmt.Errorf("failed to send response - %s", err)
                break
            }
        }
    }

    p.ctx.nsqd.logf(LOG_INFO, "PROTOCOL(V2): [%s] exiting ioloop", client)
    //正常关闭连接
    conn.Close()
    close(client.ExitChan)
    if client.Channel != nil {
        client.Channel.RemoveClient(client.ID)
    }

    return err
}
```

可以看到其实IOLoop函数主要进行有两个功能

1. p.messagePump（）对客户端订阅的消息进行获取，发送给客户端
2. p.Exec（） 对客户端发送的命令进行执行



