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



