1. ##### ksql是什么?
```
ksqldb是一个事件流数据库，
它是一种帮助你构建流处理应用程序的特殊数据库，它整合了几乎每个事件流架构中的许多组件.

```

2. ##### stream是什么?
```
ksqlDB提供两种类型的集合:流(Streams)和表(Tables)，均基于键/值对。
Streams 是不可变的,append-only的集合。添加相同key的多个事件只需附加到流尾部。
Tables 是可变的集合。只展示每个key的最新value信息。它更适于随时间变化的模型,常用于表示聚合。
```
###### stream操作:
```
//创建stream
CREATE STREAM test (profileId VARCHAR, latitude DOUBLE, longitude DOUBLE)
WITH (kafka_topic='lola-test', value_format='json', partitions=1);

//stream写数据
INSERT INTO test (profileId, latitude, longitude) VALUES ('c2309eec', 37.7877, -122.4205);

//查询stream
SELECT * FROM test
  WHERE GEO_DISTANCE(latitude, longitude, 37.4133, -122.1162) <= 1 EMIT CHANGES;
  
//删除stream
 drop stream test;
 
 SHOW streams EXTENDED;

```


3. ##### connector
```
CREATE SOURCE | SINK CONNECTOR connector_name
  WITH( property_name = expression [, ...]);
  
  
```

4. #####  table
```
CREATE TYPE <type_name> AS <type>;

SHOW tables EXTENDED;

DROP TABLE TABLE_NAME;
```

5. ##### query
```
show queries;

TERMINATE query_id;

```

6. ##### print
```
PRINT topicName [FROM BEGINNING] [INTERVAL interval] [LIMIT limit]
```



