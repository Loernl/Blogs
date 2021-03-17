##### ksql支持at-least-once(至少一次)和 exactly-once两种处理保证,在配置中启动命令
ksqlDB默认的Guarantees是at_least_once

```
processing.guarantee="at_least_once" OR
processing.guarantee="exactly-once"
```

###### at-least-once

```
配置at-least-once，记录永远不会丢失，但是可以重新传送，
如果你的流式处理失败了，则不会丢失任何数据记录也不会处理失败，
但是某些数据记录可能会被重新读取也会被重新处理。
```


###### Exactly-once 

```
记录只会被处理一次。如果ksqlDB应用程序中的生产者发送重复记录，则将其只记录一次。
Exactly-once流式处理有能力执行一次准确的读写操作。
所有处理只发生一次，包括流处理和以及流处理创建出来的materialized state，这个状态将会被流处理程序写回kafka。
```

> 使用exactly-once时必须要注意，为了实现真正的一次准确系统，消费者和生产者必须实现Exactly-once 的寓意

ksqldb如何设置Processing Guarantees？
- ksql cli
> SET 'processing.guarantee' = 'exactly_once';
- rest api

```
POST /query HTTP/1.1
Accept: application/vnd.ksql.v1+json
Content-Type: application/vnd.ksql.v1+json

{
"ksql": "SELECT * FROM pageviews EMIT CHANGES;",
"streamsProperties": {
    "processing.guarantee": "exactly_once"
}
}
```

- ksqldb configuration

```
docker run -d \
  … 
  -e KSQL_KSQL_STREAMS_PROCESSING_GUARANTEE=exactly_once \
  -e KSQL_BOOTSTRAP_SERVERS=localhost:9092 \
  … 
  confluentinc/ksqldb-server:0.14.0
```

