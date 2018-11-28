Exec\(\)用来处理客户端发送的命令，包括订阅，发送数据等操作

```go
func (p *protocolV2) Exec(client *clientV2, params [][]byte) ([]byte, error) {
    if bytes.Equal(params[0], []byte("IDENTIFY")) {
        return p.IDENTIFY(client, params)
    }
    err := enforceTLSPolicy(client, p, params[0])
    if err != nil {
        return nil, err
    }
    switch {
    case bytes.Equal(params[0], []byte("FIN")):
        return p.FIN(client, params)
    case bytes.Equal(params[0], []byte("RDY")):
        return p.RDY(client, params)
    case bytes.Equal(params[0], []byte("REQ")):
        return p.REQ(client, params)
    case bytes.Equal(params[0], []byte("PUB")):
        return p.PUB(client, params)
    case bytes.Equal(params[0], []byte("MPUB")):
        return p.MPUB(client, params)
    case bytes.Equal(params[0], []byte("DPUB")):
        return p.DPUB(client, params)
    case bytes.Equal(params[0], []byte("NOP")):
        return p.NOP(client, params)
    case bytes.Equal(params[0], []byte("TOUCH")):
        return p.TOUCH(client, params)
    case bytes.Equal(params[0], []byte("SUB")):
        return p.SUB(client, params)
    case bytes.Equal(params[0], []byte("CLS")):
        return p.CLS(client, params)
    case bytes.Equal(params[0], []byte("AUTH")):
        return p.AUTH(client, params)
    }
    return nil, protocol.NewFatalClientErr(nil, "E_INVALID", fmt.Sprintf("invalid command %s", params[0]))
}
```

p.pub是客户端向nsq投递消息

```go
func (p *protocolV2) PUB(client *clientV2, params [][]byte) ([]byte, error) {
    var err error

    if len(params) < 2 {
        return nil, protocol.NewFatalClientErr(nil, "E_INVALID", "PUB insufficient number of parameters")
    }
    //请求格式: [ PUB topicName ...]
    //判断是否是正确的topicName格式
    topicName := string(params[1])
    if !protocol.IsValidTopicName(topicName) {
        return nil, protocol.NewFatalClientErr(nil, "E_BAD_TOPIC",
            fmt.Sprintf("PUB topic name %q is not valid", topicName))
    }
    //读取消息内容
    bodyLen, err := readLen(client.Reader, client.lenSlice)
    if err != nil {
        return nil, protocol.NewFatalClientErr(err, "E_BAD_MESSAGE", "PUB failed to read message body size")
    }

    if bodyLen <= 0 {
        return nil, protocol.NewFatalClientErr(nil, "E_BAD_MESSAGE",
            fmt.Sprintf("PUB invalid message body size %d", bodyLen))
    }
    //投递的消息不得大于设置的最大消息长度
    if int64(bodyLen) > p.ctx.nsqd.getOpts().MaxMsgSize {
        return nil, protocol.NewFatalClientErr(nil, "E_BAD_MESSAGE",
            fmt.Sprintf("PUB message too big %d > %d", bodyLen, p.ctx.nsqd.getOpts().MaxMsgSize))
    }

    messageBody := make([]byte, bodyLen)
    _, err = io.ReadFull(client.Reader, messageBody)
    if err != nil {
        return nil, protocol.NewFatalClientErr(err, "E_BAD_MESSAGE", "PUB failed to read message body")
    }

    if err := p.CheckAuth(client, "PUB", topicName, ""); err != nil {
        return nil, err
    }
    //根据topicName获取topic，并投递消息
    topic := p.ctx.nsqd.GetTopic(topicName)
    msg := NewMessage(topic.GenerateID(), messageBody)
    err = topic.PutMessage(msg)
    if err != nil {
        return nil, protocol.NewFatalClientErr(err, "E_PUB_FAILED", "PUB failed "+err.Error())
    }

    return okBytes, nil
}
```

PUB 方法做一系列检查, 然后调用 topic.PutMessage\(msg\) 做具体的发送

```go
// PutMessage writes a Message to the queue
func (t *Topic) PutMessage(m *Message) error {
    //读写锁，防止冲突
    t.RLock()
    defer t.RUnlock()
    if atomic.LoadInt32(&t.exitFlag) == 1 {
        return errors.New("exiting")
    }
    //投递消息
    err := t.put(m)
    if err != nil {
        return err
    }
    //投递消息计数
    atomic.AddUint64(&t.messageCount, 1)
    return nil
}
```

t.put 把数据通过管道队里传输，如果队列已满则先保存到磁盘

```go
func (t *Topic) put(m *Message) error {
    select {
    //把消息写入chan队列
    //如果队列已满则执行default
    case t.memoryMsgChan <- m:
    default:
        //获取一个缓冲池
        //把消息写如缓冲池备份，防止数据丢失（实际上是写入磁盘）
        b := bufferPoolGet()
        err := writeMessageToBackend(b, m, t.backend)
        //把b用完后放回缓冲池
        bufferPoolPut(b)
        t.ctx.nsqd.SetHealth(err)
        if err != nil {
            t.ctx.nsqd.logf(LOG_ERROR,
                "TOPIC(%s) ERROR: failed to write message to backend - %s",
                t.name, err)
            return err
        }
    }
    return nil
}
```

在创建topic的时候就有一个线程在等待接收memoryMsgChan里面的消息，等待他的到来

在nsq/nsqd/topic.go中：

```go
// 构造一个新的topic
func NewTopic(topicName string, ctx *context, deleteCallback func(*Topic)) *Topic {
	t := &Topic{
		name:              topicName,
		channelMap:        make(map[string]*Channel),
		memoryMsgChan:     make(chan *Message, ctx.nsqd.getOpts().MemQueueSize),
		startChan:         make(chan int, 1),
		exitChan:          make(chan int),
		channelUpdateChan: make(chan int),
		ctx:               ctx,
		paused:            0,
		pauseChan:         make(chan int),
		deleteCallback:    deleteCallback,
		idFactory:         NewGUIDFactory(ctx.nsqd.getOpts().ID),
	}
	//临时的topic不需要进行磁盘保存
	if strings.HasSuffix(topicName, "#ephemeral") {
		t.ephemeral = true
		t.backend = newDummyBackendQueue()
	} else {
		dqLogf := func(level diskqueue.LogLevel, f string, args ...interface{}) {
			opts := ctx.nsqd.getOpts()
			lg.Logf(opts.Logger, opts.logLevel, lg.LogLevel(level), f, args...)
		}
		//简历一个磁盘文件保存队列，如果消息队列满了或者nsq意外结束，把未投递的消息保存到磁盘中
		t.backend = diskqueue.New(
			topicName,
			ctx.nsqd.getOpts().DataPath,
			ctx.nsqd.getOpts().MaxBytesPerFile,
			int32(minValidMsgLength),
			int32(ctx.nsqd.getOpts().MaxMsgSize)+minValidMsgLength,
			ctx.nsqd.getOpts().SyncEvery,
			ctx.nsqd.getOpts().SyncTimeout,
			dqLogf,
		)
	}
	//这里会启动一个线程用来接收pub到topic的消息
	t.waitGroup.Wrap(t.messagePump)

	t.ctx.nsqd.Notify(t)

	return t
}
```



