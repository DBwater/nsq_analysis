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



