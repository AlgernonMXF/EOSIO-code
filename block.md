# 区块 - Block
> 代码中有如下注射：       
> When a transaction is referenced by a block it could imply one of several outcomes which describe the state-transition undertaken by the block producer.     
> 当交易被区块引用时，可能意味着生产者实现了描述状态转移的结果之一


## 交易收据头transaction_receipt_header
**结构体中包含的变量**

|变量名	|类型	|含义|
|--		|--					|--|
|status		|fc::enum_type<uint8_t,status_enum>	|状态|
|cpu_usage_us	|uint32_t				|总计费CPU使用量|
|net_usage_words|fc::usigned_int			|总计费net使用量|

结构体中同时也定义了枚举类型status_eunm
```C++
enum status_enum 
{
	executed  = 0,		// succeed, no error handler executed
        soft_fail = 1,		// objectively failed (not executed), error handler executed
        hard_fail = 2, 		// objectively failed and error handler objectively failed thus no state change
        delayed   = 3, 		// transaction delayed/deferred/scheduled for future execution
        expired   = 4  		// transaction expired and storage space refuned to user
};
```

**结构体中定义的函数**
```C++
transaction_receipt_header():status(hard_fail){}
transaction_receipt_header( status_enum s ):status(s){}

//判断交易相同：仅判断状态、CPU使用、网络使用
friend inline bool operator ==( const transaction_receipt_header& lhs, const transaction_receipt_header& rhs ) 
{
	return std::tie(lhs.status, lhs.cpu_usage_us, lhs.net_usage_words) == std::tie(rhs.status, rhs.cpu_usage_us, rhs.net_usage_words);
}

fc::enum_type<uint8_t,status_enum>   status;

//total billed CPU usage (microseconds)
uint32_t                             cpu_usage_us; 

//total billed NET usage, so we can reconstruct resource state when skipping context free data... hard failures...
fc::unsigned_int                     net_usage_words; 
```
