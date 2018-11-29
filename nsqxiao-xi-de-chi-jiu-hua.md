既然是消息对列，那么我们需要考虑这样一个问题，如果nsq在某一段时间推出了，或者消息队列满了，那么消息是否就丢失了呢，显示是不太现实的，既然是消息队列，那么我就必须保证消息的可靠性

在创建topic的时候就已经考虑到了这个问题

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

2. 在channel和topic退出之前，如果有消息，没有推送完毕，则也需要对消息进行一次保存

