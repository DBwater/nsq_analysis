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

t.put是把消息投递到队列中去，如果队列满了，那么就保存消息到磁盘

```go
func (t *Topic) put(m *Message) error {
    select {
    //把消息投递到队列中去
    case t.memoryMsgChan <- m:
    default:
    //如果队列满了，那么需要一个结构来保存消息（保存到磁盘）
        b := bufferPoolGet()
        err := writeMessageToBackend(b, m, t.backend)
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

t.put是投递消息到管道，那么就有一个地方会接受这个管道过来的消息，我们可以在新建toppic的时候看到一个函数

```go
func NewTopic(topicName string, ctx *context, deleteCallback func(*Topic)) *Topic {
    t := &Topic{
        ...
    }
    ...
    //启动一个线程用来接受发送到topic的消息
    t.waitGroup.Wrap(func() { t.messagePump() })
}
```

在t.messagePump\(\)中会接受消息，然后把消息分发到topic下属的所有channel中去

```go
func (t *Topic) messagePump() {
    var msg *Message
    var buf []byte
    var err error
    var chans []*Channel
    var memoryMsgChan chan *Message
    var backendChan chan []byte

    // do not pass messages before Start(), but avoid blocking Pause() or GetChannel()
    for {
        select {
        case <-t.channelUpdateChan:
            continue
        case <-t.pauseChan:
            continue
        case <-t.exitChan:
            goto exit
        case <-t.startChan:
        }
        break
    }
    //获取topic对应的channel的状态
    t.RLock()
    for _, c := range t.channelMap {
        chans = append(chans, c)
    }
    t.RUnlock()
    if len(chans) > 0 && !t.IsPaused() {
        memoryMsgChan = t.memoryMsgChan
        backendChan = t.backend.ReadChan()
    }

    // main message loop
    for {
        select {
        //接受投递到topic的消息
        case msg = <-memoryMsgChan:
        case buf = <-backendChan:
            msg, err = decodeMessage(buf)
            if err != nil {
                t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
                continue
            }
        case <-t.channelUpdateChan:
            //Topic的channel发生变化，需要及时更新
            chans = chans[:0]
            t.RLock()
            for _, c := range t.channelMap {
                chans = append(chans, c)
            }
            t.RUnlock()
            if len(chans) == 0 || t.IsPaused() {
                memoryMsgChan = nil
                backendChan = nil
            } else {
                memoryMsgChan = t.memoryMsgChan
                backendChan = t.backend.ReadChan()
            }
            continue
        case <-t.pauseChan:
            //暂停
            if len(chans) == 0 || t.IsPaused() {
                memoryMsgChan = nil
                backendChan = nil
            } else {
                memoryMsgChan = t.memoryMsgChan
                backendChan = t.backend.ReadChan()
            }
            continue
        case <-t.exitChan:
            goto exit
        }
        //遍历topic所属的所有channel，把消息复制一份到所有的channel下
        for i, channel := range chans {
            chanMsg := msg
            // copy the message because each channel
            // needs a unique instance but...
            // fastpath to avoid copy if its the first channel
            // (the topic already created the first copy)
            if i > 0 {
                chanMsg = NewMessage(msg.ID, msg.Body)
                chanMsg.Timestamp = msg.Timestamp
                chanMsg.deferred = msg.deferred
            }
            if chanMsg.deferred != 0 {
                channel.PutMessageDeferred(chanMsg, chanMsg.deferred)
                continue
            }
            err := channel.PutMessage(chanMsg)
            if err != nil {
                t.ctx.nsqd.logf(LOG_ERROR,
                    "TOPIC(%s) ERROR: failed to put msg(%s) to channel(%s) - %s",
                    t.name, msg.ID, channel.name, err)
            }
        }
    }

exit:
    t.ctx.nsqd.logf(LOG_INFO, "TOPIC(%s): closing ... messagePump", t.name)
}
```

channel.PutMessage\(\)和topic的操作类似，就是把消息通过管道发送给channel

```go
// PutMessage writes a Message to the queue
func (c *Channel) PutMessage(m *Message) error {
    c.RLock()
    defer c.RUnlock()
    if c.Exiting() {
        return errors.New("exiting")
    }
    //投递消息
    err := c.put(m)
    if err != nil {
        return err
    }
    atomic.AddUint64(&c.messageCount, 1)
    return nil
}
```

c.put也和topic的put类似

```go
func (c *Channel) put(m *Message) error {
    select {
    case c.memoryMsgChan <- m:
    default:
        b := bufferPoolGet()
        err := writeMessageToBackend(b, m, c.backend)
        bufferPoolPut(b)
        c.ctx.nsqd.SetHealth(err)
        if err != nil {
            c.ctx.nsqd.logf(LOG_ERROR, "CHANNEL(%s): failed to write message to backend - %s",
                c.name, err)
            return err
        }
    }
    return nil
}
```

然后消息就会在

```go
func (p *protocolV2) IOLoop(conn net.Conn){
messagePumpStartedChan := make(chan bool)
...
	go p.messagePump(client, messagePumpStartedChan)
	<-messagePumpStartedChan

...
}

```

函数中被投递到客户端

