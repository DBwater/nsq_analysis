既然是消息对列，那么我们需要考虑这样一个问题，如果nsq在某一段时间退出了，或者消息队列满了，那么消息是否就丢失了呢，显示是不太现实的，既然是消息队列，那么我就必须保证消息的可靠性

nsq里面有一个结构体保存着nsq所有的topic和channel的相关信息：

```go
type meta struct {
    Topics []struct {
        Name     string `json:"name"`        //topic名字
        Paused   bool   `json:"paused"`        //topic状态
        Channels []struct {
            Name   string `json:"name"`    //channel名字
            Paused bool   `json:"paused"`    //channel状态
        } `json:"channels"`
    } `json:"topics"`
}
```

当nsq退出的时候会把相关的数据写入到磁盘中

```go
func (n *NSQD) Exit() {
    ...
    n.Lock()
    err := n.PersistMetadata()
    if err != nil {
        n.logf("ERROR: failed to persist metadata - %s", err)
    }
    n.logf("NSQ: closing topics")
    for _, topic := range n.topicMap {
        topic.Close()
    }
    n.Unlock()
    ...
}
```

1. PersistMetadata\(\)函数就是把nsq的相关信息格式化为json写入到磁盘中，主要是topics和channels的名字和状态。
2. topic.Close\(\)的时候也会进行一个flush，把topic中的数据刷新到磁盘中去，只不过是用的另外一种数据结构保存的。

```go
func (n *NSQD) PersistMetadata() error {
    // persist metadata about what topics/channels we have, across restarts
    //保存topics和channels原数据，防止程序结束或者重启时导致的数据丢失
    fileName := newMetadataFile(n.getOpts())
    // old metadata filename with ID, maintained in parallel to enable roll-back
    //fileNameID文件是fileName文件的软连接，内容和fileName一样，用于回滚操作
    fileNameID := oldMetadataFile(n.getOpts())

    n.logf(LOG_INFO, "NSQ: persisting topic/channel metadata to %s", fileName)
    //循环遍历得到所有topics和channels的名称和状态，并格式化为json字符串
    js := make(map[string]interface{})
    topics := []interface{}{}
    for _, topic := range n.topicMap {
        if topic.ephemeral {
            continue
        }
        topicData := make(map[string]interface{})
        topicData["name"] = topic.name
        topicData["paused"] = topic.IsPaused()
        channels := []interface{}{}
        topic.Lock()
        for _, channel := range topic.channelMap {
            channel.Lock()
            if channel.ephemeral {
                channel.Unlock()
                continue
            }
            channelData := make(map[string]interface{})
            channelData["name"] = channel.name
            channelData["paused"] = channel.IsPaused()
            channels = append(channels, channelData)
            channel.Unlock()
        }
        topic.Unlock()
        topicData["channels"] = channels
        topics = append(topics, topicData)
    }
    js["version"] = version.Binary
    js["topics"] = topics

    data, err := json.Marshal(&js)
    if err != nil {
        return err
    }

    tmpFileName := fmt.Sprintf("%s.%d.tmp", fileName, rand.Int())
    //将格式化后的json字符串写入到文件中，并重命名文件
    err = writeSyncFile(tmpFileName, data)
    if err != nil {
        return err
    }
    err = os.Rename(tmpFileName, fileName)
    if err != nil {
        return err
    }
    // technically should fsync DataPath here

    stat, err := os.Lstat(fileNameID)
    if err == nil && stat.Mode()&os.ModeSymlink != 0 {
        return nil
    }

    // if no symlink (yet), race condition:
    // crash right here may cause next startup to see metadata conflict and abort

    tmpFileNameID := fmt.Sprintf("%s.%d.tmp", fileNameID, rand.Int())

    if runtime.GOOS != "windows" {
        err = os.Symlink(fileName, tmpFileNameID)
    } else {
        // on Windows need Administrator privs to Symlink
        // instead write copy every time
        err = writeSyncFile(tmpFileNameID, data)
    }
    if err != nil {
        return err
    }

    err = os.Rename(tmpFileNameID, fileNameID)
    if err != nil {
        return err
    }
    // technically should fsync DataPath here

    return nil
}
```

topic.Close\(\)会进行数据文件的保存。

在创建topic的时候就已经考虑到了数据文件持久化的问题，topic和channel的数据保存是通过一个diskqueue的结构体来保存的。

```
func NewTopic(topicName string, ctx *context, deleteCallback func(*Topic)) *Topic {
...
        //创建一个备份的结构体，用来持久化保存消息，防止消息的丢失
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

...
}
```

diskqueue在 github.com/nsqio/go-diskqueue/文件中

在topic和channel中都有用到这个结构体来保存数据

1. 在/nsq/nsqd/topic和nsq/nsqd/channel中如果消息队列满了，那么就需要先把消息存储到t.backend中去

```go
func (t *Topic) put(m *Message) error{
...
select {
    //把消息投递到队列中去
    case t.memoryMsgChan <- m:
    default:
    //如果队列满了，那么需要一个结构来保存消息（保存到磁盘）
    b := bufferPoolGet()
    err := writeMessageToBackend(b, m, t.backend)
    bufferPoolPut(b)
    t.ctx.nsqd.SetHealth(err)
}

func (c *Channel) put(m *Message) error {
...
select {
    case c.memoryMsgChan <- m:
    default:
    //如果队列满了，那么需要一个结构来保存消息（保存到磁盘）
    b := bufferPoolGet()
    err := writeMessageToBackend(b, m, c.backend)
...
}
```

1. 在channel和topic退出之前，如果有消息，没有推送完毕，则也需要对消息进行一次保存，当topic和channel退出之前会调用flush\(\)函数，把数据全部刷到磁盘中保存起来，防止消息的丢失

```go
func (c *Channel) flush() error {
    var msgBuf bytes.Buffer

    if len(c.memoryMsgChan) > 0 || len(c.inFlightMessages) > 0 || len(c.deferredMessages) > 0 {
        c.ctx.nsqd.logf(LOG_INFO, "CHANNEL(%s): flushing %d memory %d in-flight %d deferred messages to backend",
            c.name, len(c.memoryMsgChan), len(c.inFlightMessages), len(c.deferredMessages))
    }

    for {
        select {
        case msg := <-c.memoryMsgChan:
        //如果消息队列中还有数据没有发送，则保存到磁盘
            err := writeMessageToBackend(&msgBuf, msg, c.backend)
            if err != nil {
                c.ctx.nsqd.logf(LOG_ERROR, "failed to write message to backend - %s", err)
            }
        default:
            goto finish
        }
    }

finish:
    c.inFlightMutex.Lock()
    for _, msg := range c.inFlightMessages {
    //如果有正在投递的消息（已发送，但没有得到响应）。也保存到磁盘
        err := writeMessageToBackend(&msgBuf, msg, c.backend)
        if err != nil {
            c.ctx.nsqd.logf(LOG_ERROR, "failed to write message to backend - %s", err)
        }
    }
    c.inFlightMutex.Unlock()

    c.deferredMutex.Lock()
    for _, item := range c.deferredMessages {
        msg := item.Value.(*Message)
        //延迟投递的消息，目前还未投递，也保存到磁盘
        err := writeMessageToBackend(&msgBuf, msg, c.backend)
        if err != nil {
            c.ctx.nsqd.logf(LOG_ERROR, "failed to write message to backend - %s", err)
        }
    }
    c.deferredMutex.Unlock()

    return nil
}
```

writeMessageToBackend\(\)函数在/nsqd/message.go文件里面,首先会把消息写入到buffer缓冲区中，然后放到BackendQueue中持久化保存

```go
func writeMessageToBackend(buf *bytes.Buffer, msg *Message, bq BackendQueue) error {
    buf.Reset()
    _, err := msg.WriteTo(buf)
    if err != nil {
        return err
    }
    return bq.Put(buf.Bytes())
}
```

msg实现了接口WeiterTo\(\),把消息写入buffer缓冲区，然后用BackendQueue保存下来

