##### Time

```
时间分为事件的时间，写入kafka的时间以及处理的时间
```
- 事件时间

```
事件时间是写入数据源的时间，即将producer生成的时间戳。
如果topic的 message.timestamp.type=CreateTime，则以下三种情况下都会被定义为事件时间
```
1. 创建producer record时，默认情况下它不包含时间戳
2. producer在record上明确的设置时间戳
3. 如果在producer调用send（）方法时未设置时间戳，则当前wall-clock时间将自动设置被record的时间戳

- 摄取时间、

```
即kafka将记录存储到topic的时间，producer发送数据到topic，topic record默认生成的时间戳。如果事件时间与摄取时间之前误差很小，那么kafka摄取时间=~ 事件事件
```
- 处理时间

```
流处理应用程序使用记录的时间。处理时间在摄取时间之后发生，可以立即发生也可以延迟发生
```

##### kafka record的timestamp是如何生成的？

```
record中的timestamp可以由producer控制，可以由kafka server来控制，
这取决于你在topic中的配置项:message.timestamp.type
```
> message.timestamp.type,此配置项有两个可选配置时间:[CreateTime,LogAppendTime],默认的配置为CreateTime

```
CreateTime:broker会使用producer生成的时间戳来设置事件时间
LogAppendTime：当record被添加到topic的log时，
broker会使用它服务器的时间覆盖record的timestamp，
当配置了LogAppendTime时，producer设置的timestamp无效。
```
##### ksqldb输出的timestamp如何计算
- ksqldb输出的timestamp使用规则如下:

1. 如果是直接处理输入记录来生成新的输出记录时，输出记录的时间戳将和输入记录的时间戳相同
2. 如果是通过函数处理生成新的输出记录时，输出记录的时间戳是当前流任务的当前内部事件
3. 但是对于无状态的一些操作，输出记录是输入记录的时间戳，比如flatMap，所有输出记录的timestamp和输入记录的timestamp保持一致，

- 对于经过聚合和join的操作，timestamp使用的规则如下:
1. 进行join的流或者表，输出记录的时间戳计算为:max(left.ts, right.ts)
2. 如果是流和表之间进行join，输出记录的时间戳记为流记录中的时间戳
3. 对于聚合，结果更新记录的时间戳取自触发更新的最新输入记录
4. 对于聚合，最大时间戳是针对每个键在所有记录上计算的。即窗口输出记录的时间戳是在记录上计算的当前时间。

