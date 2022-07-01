重点是记录，供商业分析决策，开源的可观测性的美好图景优先级排后面，比如 metric-logs-tracing 联动。

## Background
从使用者的角度：用户需要了解系统的负载情况、开发需要系统运行情况为优化提供数据依据。
从产品的角度：竞品的基本功能，我们也得有呀，而且要做的更好。

分布式系统的关联分析
-   不同数据指标的关联分析需求
    -   例如：A指标的P99=100ms、B指标的P99=50ms，是否因为B指标影响了A指标？
    -   例如：一个用户的请求，需要经过多个函数运算，经过多个组件并行或串行执行。当验证执行链路是否正常，或对执行链路进行优化的时候，就需要对系统非常熟悉。如何进行问题诊断和根因溯源就需要非常高的门槛。
-   对单一样例进行分析的需求
    -   运行场景本身很复杂，例如：多租户混用、多种工作负载同时运行。
    -   metric波动小。例如：原来执行时间很短的操作变慢了，但由于执行次数、耗时绝对值小等等问题，无法对 AVG/MAX/P99等指标产生影响。

## Instrument

以 tracing 为基础，拓展携带 sqlstats、metric、logs

复用 otel 用作 instrument 能满足当前需求，主要包括`TraceProvider`, `Tracer`, `Span`
```go
// startup init
// 创建 Exporter, 负责将span数据输出值backend，在MO是负责写入到 MO Database
exp, err := newExporter(f)

// 初始化 TraceProvider, 可注册：Exporter、Resource、IDGenerator、Sampler等
tp := trace.NewTracerProvider(
    trace.WithBatcher(exp),
    trace.WithResource(newResource()),
)
otel.SetTracerProvider(tp)
// END> startup init

func ExampleFunc() {
   // 创建一个 span，并记录开始时间
   _, span := otel.Tracer(""/*instrumentationName*/).Start(ctx, "foo"/*Span Name*/)

  err := errors.New("new error msgs")
  // optional: 记录error信息
  span.RecordError(err)
  // optional: 设置Span Status
  span.SetStatus(codes.Error, err.Error())
  // optional: 记录metric相关数据
  span.SetAttributes(attribute.Int64("test_cost", int64(123)))
  // 结束Span
  span.End()
}
```


Json样例：
主要由 4部分构成：
1. 由Name、SpanContext、Parent、StartTime、EndTime 构成了Span的 基本信息
2. Attributes，则记录Span生命周期中发生动作，先用户记录metric相关的统计
3. Events，记录Span的 log 和 error 
4. Resource，记录Span所在server 的静态信息，例如节点ID、server类型、server版本等。

```json
{
  "Name":"foo", 		// otel.Tracer(..).Start(ctx, ""/*Name*/)
  "SpanContext":{
	  "TraceID":"ce8774caf582a052fa08e360415cbfff", // 来自 ctx 中记录的 TraceID
      "SpanID":"3835a5290f69e508",					// 由 otel.Tracer(..).Start(...) 调用 IDGenerator 生成
      "TraceFlags":"01",
      "TraceState":"",
      "Remote":false},
  "Parent":{			// 来自 ctx 记录的父亲的Span信息
	  "TraceID":"ce8774caf582a052fa08e360415cbfff",
      "SpanID":"1a9d64a4ebea48bf",
      "TraceFlags":"01",
      "TraceState":"",
      "Remote":false},
  "SpanKind":1,
  "StartTime":"2022-03-24T10:43:47.682053+08:00",
  "EndTime":"2022-03-24T10:43:47.682054213+08:00",
  "Attributes":[{"Key":"test_cost","Value":{"Type":"INT64","Value": 123}}], // For ComponentsStats 封装
  "Events":[{
			"Name": "HE SAYS",
			"Attributes": [{"Key": "answer","Value": {"Type": "INT64","Value": 42}}],
			"DroppedAttributeCount": 0,
			"Time": "2022-03-24T10:43:47.6820533+08:00"
		}, // For logs
		{
            "Name": "exception",
            "Attributes": [{"Key": "exception.type","Value": {"Type": "STRING","Value": "*errors.errorString"}},
                {"Key": "exception.message","Value": {"Type": "STRING","Value": "new error msgs"}}
            ],
            "DroppedAttributeCount": 0,
            "Time": "2022-03-24T10:43:47.682054213+08:00"
		} // For error record
	], 
  "Links":null,
  "Status":{"Code":"Unset","Description":""},
  "DroppedAttributes":0,
  "DroppedEvents":0,
  "DroppedLinks":0,
  "ChildSpanCount":0,
  "Resource": [			// 来自 TracerProvider 注册的 Resource 信息
        {"Key":"service.name",          "Value":{"Type":"STRING","Value":"example"}},
        {"Key":"service.version",       "Value":{"Type":"STRING","Value":"v0.1.0"}},
	    {"Key":"telemetry.sdk.language","Value":{"Type":"STRING","Value":"go"}},
	    {"Key":"telemetry.sdk.name",    "Value":{"Type":"STRING","Value":"opentelemetry"}},
	    {"Key":"telemetry.sdk.version", "Value":{"Type":"STRING","Value":"1.7.0"}}
    ]
}
```

Table  存储结构概要
- Plan A）使用json对Attributes 和 Resources 进行存储，表结构类似

  | Primary Key      | Span Name | TraceId... | Start Time | End Time | Resources         | Attributes        | Events            |
  | ---------------- | --------- | ---------- | ---------- | -------- | ----------------- | ----------------- | ----------------- |
  | Auto increment  | foo       | - | - | - | {Key: Value, ...} | {Key: Value, ...} | {Key: Value, ...} |

- Plan B）不支持json，就需要单独 key，value进行存储了（如下表展示 Attributes）

  | Primary Key      | Span Name | TraceId... | Start Time | Duration | Attributes.Key | Attributes.Value |
  | ---------------- | --------- | ---------- | ---------- | -------- | -------------- | ---------------- |
  | Auto increment  | foo       | - | - | - | test_cost      | 123              |
  | - | foo       | - | - | - | key_2          | value_2          |


 | Primary Key      | SQL | Text | Start Time | Duration | Attributes.Key | Attributes.Value |
  | ---------------- | --------- | ---------- | ---------- | -------- | -------------- | ---------------- |
  | Auto increment  | foo       | - | - | - | test_cost      | 123              |
  | - | foo       | - | - | - | key_2          | value_2          |


还是有一些需要解决或优化的问题的：
- P0: `context.Context` 传输`TraceID/SpanID/ParentSpanID`等信息
- P0: gen random ids - crdb 有 fastInt63 @pkg/util/randutil/rand.go:122
- P0: read sys time - tidb 有 tikv 的高效 tracing
- P0: batchProcessor 的 channel 选择和 exporter 的唤醒策略，[Optimizing OpenTelemetry’s Span Processor for High Throughput and Low CPU Costs](https://doordash.engineering/2021/04/07/optimizing-opentelemetrys-span-processor/)
- P1: span 的分配 - crdb 做一个 pool 减少内存分配
- P1: events 和 attrs buffer 的使用回收，~~以及满容量后的 drop 是 O(n)~~
- P1: span 内存占用，最小化



### 1. sqlstats

- 每个算子至少一个 span，算子close 时调用 RecordComponentStats(ComponentStats) 接口，产生一份 Stats，序列化后存到 `Span.Attributes`
- Stats 分为两个部分，一是基础信息，内存网络磁盘，二是独立信息，各个算子不同，自行定义，用 protobuf oneof 表示
- Run 之前有一个名为 tracedStmt 的 root span，记录 parseTime、ServiceTime、Plan 等查询的全局信息

```protobuf
OperatorStats {
	KeyValues() []KeyValue
}

ComponentsStats {
    enum Type
    uint32 NodeID
    uint32 QueryID
    uint64 ProcessedRows
    ExecTime {
        uint64 Setup
        uint64 Processing // CPU
    }
    uint64 MemoryUsage
    Network {  
        uint64 BytesSent
        uint64 RowsSent 
        uint64 BytesReceived
        uint64 RowsReceived
        uint64 SerializationTime
        uint64 DeserializationTime
    }
    Disk {
        uint64 DiskWrite
        uint64 DiskRead   
    }
    
    oneof Info {
       XxxInfo 
       YxxInfo
       ...
    }
}

TracedStmtInfo {
    uint64 ServeLat
    uint64 ParseLat
    uint64 PlanLat
    uint64 RunLat
    uint64 OverheadLat
    string PlanDigest // 后续用来组织算子关系
    ...
}

TableScanInfo {
    uint32 ScanSegs
    uint32 ScanBlocks
    uint64 FilteredRows
    repeated string Columns
    ...
}
```


### 2. logs
日志记录到Span中，可方便定位分析单一SQL请求在多个组件中的执行过程。
当前MO的loggerutil实现功能相对简洁，可考虑进行如下丰富：增加 prefix的日志信息，可与Span关联数据QueryID相关信息。

此处有2 个分歧：
- Plan A，新增`Eventf(context.Context, formatter string, args ...any)`单独支持Log 与Span关联，仅指定的日志上报
- Plan B，所有日志统一都上报。

> 在 crdb 里用的是 `eventInternal`，使用接口 `log.Eventf`，用于将log记录到 Span中，也就是并非每个日志都进入到 span
> 因为当前span信息需从 context 中拿取，log 需要 context。
> 这个在 crdb 和 tidb 中都用，比如 crdb，还利用 context 自动添加 prefix，可以明确日志的模块和逻辑链。

Plan A）函数接口，专用于log记录到Span
```go
log.Eventf(ctx, "trying to remove %s", id)
```
Plan B）默认所有日志都记录到Span中
```go
log.Infof(ctx, "info message formatter", args...)
log.Errorf(ctx, "error message formatter", args...)
...
```

在Span中记录的数据数据结构
```
type Event struct {
	Name string
	Attributes []KeyValue
	Time time.Time
	DroppedAttributeCount int
}
```


### 3. metrics

grafana corelate metrics and logs by labels. Nothing new.

用 metric exemplars 包含 tracing id。接口比较固定，怎么传 ctx？

或者在 sql 执行单独存一些 metric。


```
for {
	case <- timeer():
		span = trace.Start()
		span.SetAtr(CPU, )
}
```


## Export

BatchProcessor + MO-exporter，处理 batch buffer、补充 metadata、分离 stats 类型、写入 S3


每个节点采集 span 后，在本地进行缓存，当缓存超过一定阈值、或超过时间阈值后将数据写入库表中。节点通过调用mo的批量写入接口将数据写入到S3，批量写入大致过程为：数据持续写入至S3, 完成后通过 commit提交整个文件。
![[otel-stats.excalidraw|800]]

```![[otel-stats2.excalidraw|1430]]```










## P0 Case ?
1. Execution Latency By Phase (like CRDB)
    <img src="file:///Users/jacksonxie/Nutstore Files/我的坚果云/workspace/action mo/execution_latency.png" alt="image-20220624095010" style="zoom:40%;" />
2. Query Profile (like Snowflake)
    <img src="file:///Users/jacksonxie/Nutstore Files/我的坚果云/workspace/action mo/function_all_chain.png" alt="image-20220624095010" style="zoom:30%;" />
