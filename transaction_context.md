# transaction_context - 交易环境
> 发送/完成交易之前，预先配置环境           
>           

**类中包含的变量**

|变量名		|类型				|大小	|含义|
|--			|--				|--	|--|
|controller_impl	|friend struct			|	|控制器实现|
|apply_context		|friend class			|	|应用环境|
|control		|controller&			|	|控制器|
|trx			|const signed_transaction	|	|已签名的交易|
|id			|transaction_id_type		|	|交易id|
|undo_session		|chainbase::database::session	|	|会话|
|trace			|transaction_trace_ptr		|	||
|start			|timt_point			|	|开始时间|
|published		|time_pont			|	|发布时间|
|executed		|vector<action_receipt>		|	|已被接受的action|
|bill_to_accounts	|flat_set<account_name>		|	|账户账单|
|validate_ram_usage	|flat_set<account_name>		|	|账户可用ram数量|
|initial_max_billable_cpu |unit64_t			|	|初始最大可计费cpu，默认为0|
|delay			|microseconds			|	|延时，单位：毫秒|
|is_input		|bool				|	|初始化为false|
|apply_context_free	|bool				|	|应用环境是否收费，默认不收费|
|can_subjectively_fail	|bool				|	|可在主观上失败？，默认为真|
|deadline		|time_point			|	|截止日期？，默认为最大时间|
|leeway			|microseconds			|	|余地，默认为3000ms|
|billed_cpu_time_us	|int64_t			|8B	|可计费cpu时长|
|is_initialized		|bool				|	|判断是否已经初始化，默认为无|
|net_limit		|uint64_t			|8B	|网络限制，默认为0|
|net_limit_due_to_block	|bool				|	|区块判断的网络限制，默认为真|
|eager_net_limit	|uint64_t			|8B	|默认为0|
|net_usage		|uint64_t&			|8B	|网络使用量|
|objective_duration_limit|microseconds			|	|客观限制|
|_deadline		|time_point			|	|截至日期|
|deadline_exception_code|int64_t			|	|截至日期异常代码|
|billing_timer_exception_code|int64_t			|	|计费定时器异常代码|
|pseudo_start		|time_point			|	|伪开始|
|billed_time		|microseconds			|	|计费时间|
|billing_timer_duration_limit|microseconds		|	|计费定时器的持续时间限制|
                 
**类中包含的函数**

* 初始化网络使用量
```C++
void init( uint64_t initial_net_usage );           
```
* 初始化交易
```C++
//初始化隐式交易：网络使用量初始化为0
void init_for_implicit_trx( uint64_t initial_net_usage = 0 );
void init_for_input_trx( uint64_t packed_trx_unprunable_size,uint64_t packed_trx_prunable_size,uint32_t num_signatures); 
//初始化延期交易
void init_for_deferred_trx( fc::time_point published );
```
* 
```C++ 
void exec();
void finalize();
void squash();
```
* 增加资源
```C++
inline void add_net_usage( uint64_t u );			//增加网络使用量
void add_ram_usage( account_name account, int64_t ram_delta );	//增加ram使用量
```
* 检验
```C++
void check_net_usage()const;	//检查网络使用量
void checktime()const;		//检查时间
```
* 暂停/继续计时器
```C++
void pause_billing_timer();	//暂停计费计时器
void resume_billing_timer();	//继续
```
* 分发action
```C++
void dispatch_action( action_trace& trace, const action& a, account_name receiver, bool context_free = false, uint32_t recurse_depth = 0 );
inline void dispatch_action( action_trace& trace, const action& a, bool context_free = false );
```
* 调度交易
```C++
void schedule_transaction();
```
* 记录交易
```C++
void record_transaction( const transaction_id_type& id, fc::time_point_sec expire );
```
* 验证CPU使用情况以计费
```C++
void validate_cpu_usage_to_bill( int64_t u, bool check_minimum = true )const;
```

### 代码
```C++
//构造函数
transaction_context::transaction_context( controller& c,            
					const signed_transaction& t,               
					const transaction_id_type& trx_id,             
					fc::time_point s )
:control(c),trx(t),id(trx_id),undo_session(c.db().start_undo_session(true))
,trace(std::make_shared<transaction_trace>()),start(s)
,net_usage(trace->net_usage),pseudo_start(s)
{
	trace->id = id;
      	executed.reserve( trx.total_actions() );
      	FC_ASSERT( trx.transaction_extensions.size() == 0, "we don't support any extensions yet" );
}



```
