[TOC]

# 背景

从使用者的角度：用户需要了解系统的负载情况、开发需要系统运行情况为优化提供数据依据。

从产品的角度：竞品的基本功能，我们也得有呀，而且要做的更好。

分布式系统的关联分析

---

- 不同数据指标的关联分析需求
  - 例如：A指标的P99=100ms、B指标的P99=50ms，是否因为B指标影响了A指标？
  - 例如：一个用户的请求，需要经过多个函数运算，经过多个组件并行或串行执行。当验证执行链路是否正常，或对执行链路进行优化的时候，就需要对系统非常熟悉。如何进行问题诊断和根因溯源就需要非常高的门槛。

- 对单一样例进行分析的需求
  - 运行场景本身很复杂，例如：多租户混用、多种工作负载同时运行。
  - metric波动小。例如：原来执行时间很短的操作变慢了，但由于执行次数、耗时绝对值小等等问题，无法对 AVG/MAX/P99等指标产生影响。

## 需求点

1. tracing要做到函数级别；
2. 开发可任意添加traceing数据
3. tracing主要通过异步批处理的方式统计数据；
4. metric、tracing、log需要相互关联

==>

1. 获取当前文件名与行号

2. 数据通过存储引擎的批处理接口写入数据。（外部依赖：存储引擎需提供批处理接口）

3. 同tracing关联的metric 和统计聚合的 metric可以是两种统计类型？

4. tracing主要有3部分构成：发起者+执行者+统计信息

   - 发起者有两类：1）用户触发的SQL；2）系统自驱动
   - 执行者：节点+函数+对象ID/GoroutingID
   - 统计信息：统计对象、开始时间、结束时间、操作消耗（内存、IO、网络延迟等）、key操作范围等。

   

# 整体架构



# 数据结构

```
{
  sql_identify {
  	session_id
  	transaction_id
  	query_id
  	statement_text
  	statement_fingerprint
  }
	trace_identify {
		trace_id
		span_id
		parent_span_id
	}
	resources {
		role
		node_id // id or node_ip
	}
	name
	start_time_unix_nano
	end_time_unix_nano
	attributes { // "key": value, "key2": value, ...
		"IO": 1,
		"rows": 1,
		"keys": [start_key, end_key],
	},
	event: { // 可用于记录 log信息
		start_time_unix_nano,
		name,
		attributes {
			"context": "",
			"level": "debug",
		},
	}
}
```





# 接口定义

目前暂定使用`OpenTelemetry` 接口，需进一步分析定位是否能否满足mo的场景



# P0目标

## tracing采集

## SqlStats 统计



# 性能瓶颈

## span id/trace id生成器

目前使用 OpenTelemetry 默认生成器，每次生成与 `1.095~1.104 us`

假设QPS 为 60w，则平均延迟为：`1.67 us`

## 批量插入性能

1) storage engin 
2) Project



中心节点？
