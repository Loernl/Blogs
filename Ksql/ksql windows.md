##### windows分为
- ###### tumbling windows(翻滚窗口，不重叠),
- ###### hopping windows（重叠窗口，跳跃窗口）,
- ###### session windows（动态大小，不重叠的数据驱动窗口。会话窗口是根据传入数据动态调整大小的，并由活动间隔（由不活动的间隙分隔）定义）
 
| Window type     | Behavior      | Description                    |
| --------------- | ------------- | ------------------------------ |
| Tumbling Window | Time-based    | 固定时间，不重叠，无间隙的窗口 |
| Hopping Window  | Time-based    | 固定持续时间的重叠窗口 |
| Session Window  | Session-based | 动态大小，不重叠的数据驱动窗口 |

---
Hopping windows,跳跃窗口，取决于窗口大小以及窗口跳跃间隔，因为跳跃窗口是可以重叠的，所以一条记录可以属于多个窗口，在跳跃窗口中，窗口的开始时间是包括在内的，结束时间是不包括在内的。
```
// 根据记录的时间戳将输入记录分为固定大小的窗口，可能会重叠
SELECT WINDOWSTART, WINDOWEND, aggregate_function
  FROM from_stream
  WINDOW HOPPING (SIZE 30 SECONDS, ADVANCE BY 10 SECONDS)
  EMIT CHANGES;
```

---
 Tumbling，滑动窗口是一种固定大小，不重叠，无间隙的窗口。它只取决于窗口的持续时间。也是一种特殊的跳跃窗口，当跳跃窗口的持续时间==前进间隔时，它就是滑动窗口，因为滑动窗口是不重叠的，所以每一条记录只在一个窗口中出现。滑动窗口之前都是相邻的，所以一个窗口的结束标志着另一个窗口的开始。
 滑动窗口也是包含开始时间，不包含结束时间。这对于非重叠窗口很重要，在非重叠窗口中，每条记录必须恰好包含在一个窗口中。
```
//根据记录的时间戳将输入记录分组为固定大小的，不重叠的窗口
SELECT WINDOWSTART, WINDOWEND, aggregate_function
  FROM from_stream
  WINDOW TUMBLING (SIZE 5 SECONDS)
  EMIT CHANGES;
```

---
session：会话窗口，它代表一个活动周期，由指定的不同活动间隙分开。在现有会话不活动间隙内发生的带有任何时间戳的记录都会被合并到现有会话中，如果记录的时间戳发生在会话间隙之外，则会新建一个新会话。如果最后来的记录时间比指定的不活动间隔还早，就会启动一个新的会话窗口。它与其他会话窗口之前的不同在于:
###### - ksqldb跨key独立跟踪所有会话窗口，所以不同的key有不同的开始和结束时间
###### - 会话窗口的持续时间有所不同，即使在同key的窗口通常也具有不同的持续时间
- 会话窗口的开始时间和结束时间都会包含。一个会话窗口至少有一条记录，不可能有0条记录。<br/>
- 如果会话窗口仅包含一条记录，则该记录的row time时间戳与该窗口的开始时间和结束时间相同，可以通过 WINDOWSTART and WINDOWEND两个系统自带的列来访问记录 <br/>
- 如果>=2条记录,则最早记录的row time时间与WINDOWSTART相同，最新的记录的row time时间与WINDOWEND时间相同
```
// 将具有相同键的输入记录分组到一个窗口中，以进行聚合和联接之类的操作
SELECT WINDOWSTART, WINDOWEND, aggregate_function
  FROM SESSION (60 SECONDS) //时间间隔为60s的会话窗口在
  WINDOW window_expression
  EMIT CHANGES;
```

##### window retention
```
每一种windows，都可以配置ksqldb保留的过去的窗口，这个大小必须大于窗口大小和任何宽限期的总和,表示数据在table中存在的时间，决定结果topic的ttl
```
```
eg:
  CREATE TABLE pageviews_per_region AS
  SELECT regionid, COUNT(*) FROM pageviews
  WINDOW HOPPING (SIZE 30 SECONDS, ADVANCE BY 10 SECONDS, RETENTION 7 DAYS, GRACE PERIOD 30 MINUTES)
  WHERE UCASE(gender)='FEMALE' AND LCASE (regionid) LIKE '%_6'
  GROUP BY regionid
  EMIT CHANGES;
```