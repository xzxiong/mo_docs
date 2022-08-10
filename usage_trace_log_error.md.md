# 背景
需求：

# Code
- trace
- log
	- logutil/api.go
	- logutil/loguilt2/api.go
- error
	- pkg/util/errors
	- pkg/common/moerr

example as: [pkg/util/trace/example](https://github.com/xzxiong/matrixone/blob/trace_testcase/pkg/util/trace/example/main.go) 
## error code/message
Code [pkg/common/moerr](https://github.com/xzxiong/matrixone/blob/trace_testcase/pkg/common/moerr/error.go)
1. define `ErrXxxZzzz` const
2. regist `ErrXzzZzzz: {MOErrorCode, messageFormatter, MysqlErrorCode}`  in `errorMsgRefer` map
3. use `New(MOErrorCode, args ...any)` or `NewWith(context.Context, MOErrorCode, args ...any)` get error instance

- **Error Code Format**

```
integer:   10 000
           -- ---
      分类编号  递增编号
```




# Data
table: `statement_info`, `span_info`, `log_info`, `error_info`
