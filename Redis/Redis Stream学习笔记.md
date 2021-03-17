##### 1. Stream
```
Stream就是一个消息链表，将所有加入的消息都串起来，每个消息都会有一个唯一名称（Redis的key），它在首次调用xadd指令追加消息时自动创建，消息是持久化的，即使redis重启，消息都还会在。
而每个stream都可以挂载多个消费组，消费组会有游标 last_delivered_id 在stream数组上往前滑动，表示当前消费组消费到的消息位置。每个消费组都有一个stream内唯一的名称，消费组不会自动创建，它需要使用单独的指令(xgroup )进行创建，需要指定从stream的某一个消息ID进行消费。
而每个消费组的状态都是独立的，相互不受影响。而同一个消费组可以挂载多个消费者，这些消费者之前是竞争的关系，任何一个消费组读取了消息都会使得游标往前滑动。
这种实现和kafka是相似的，都有消费组和消费者，而kafka中是使用offset来表示消费的游标。

```
###### 2. Stream如何保证消费不会丢失?
```
Stream是如何确保客户端至少消费了消息一次而不是在网络传输的中途中丢失了呢？
Stream是通过Pending Entries List这个数据结构来保证的，每个消费者的内部都会有个状态变量pending_ids，它记录了当前已经被客户端读取的消息，但是还没有ack，如果客户端没有ack，这个变量里面的消息id会越来越多，一旦某个消息被ack，它就开始减少，这个变量在redis官方被称为PEL。
```

###### 3. PEL如何保证消息不会丢失?
```
PEL主要保存消费组的消息pending_ids，用户在消费stream中的消息时，redis服务器将消息回复给客户端的过程中，客户端突然断开了连接，消息就会丢失。
但是PEL里已经保存了发出去的消息ID，客户端重新连接之后，可以再次收到PEL的消息ID列表，但是此时消费的起始消息ID不能为参数>,而必须是任意有效的ID，可以设置为0-0，表示读取所有的PEL消息以及自last_delivered_id之后的新消息.
```

###### stream commands
```
xadd 追加消息
// * 号表示服务器自动生成 ID，后面顺序跟着一堆 key/value 
eg: xadd codehole * name laoqian age 30 
```
```
xdel 删除消息，这里的删除仅仅是设置了标志位，不影响消息总长度
eg:xdel codehole 1527849609889-0 
```
```
xrange 获取消息列表，会自动过滤已经删除的消息 
// 指定最大消息 ID 的列表 
eg:xrange codehole - 1527849629172-0 
```

```
xlen 消息长度
eg:xlen codehole 
```
```
del 删除 Stream
// 删除整个stream
eg:del codehole 
```

```
xread 独立消费
// 从stream头部读取两条消息
eg:xread count 2 streams codehole 0-0 
// 从 Stream 尾部读取一条消息，毫无疑问，这里不会返回任何消息
eg: xread count 1 streams codehole $ 
// 从尾部阻塞等待新消息到来，下面的指令会堵住，直到新消息到来
eg:xread block 0 count 1 streams codehole $
// 我们从新打开一个窗口，在这个窗口往 Stream 里塞消息
eg:xadd codehole * name youming age 60
```

```
xgroup 创建消费组
// 创建消费组并表示从头开始消费 
eg: xgroup create codehole cg1 0-0 
// $ 表示从尾部开始消费，只接受新消息，当前 Stream 消息会全部忽略
eg: xgroup create codehole cg2 $
// 获取 Stream 信息 
eg: xinfo stream codehole 
// 获取 Stream 的消费组信
eg: xinfo groups codehole
```

```
xreadgroup 消费组组内消费
//  > 号表示从当前消费组的 last_delivered_id 后面开始读 
//  每当消费者读取一条消息，last_delivered_id 变量就会前进
eg:xreadgroup GROUP cg1 c1 count 1 streams codehole > 
// 观察消费组信息
eg:xinfo groups codehole 
// 如果同一个消费组有多个消费者，我们可以通过 xinfo consumers 指令观察每个消费者的状态 
eg: xinfo consumers codehole cg1 
// ack一条消息
eg: xack codehole cg1 1527851486781-0 
// ack所有消息
eg：xack codehole cg1 1527851493405-0 1527851498956-0 1527852774092-0 1527854062442-0
```