# producer_plugin

## 区块生产

#### 流程：
* producer_plugin
	* plugin_startup()
		* 获取lib->id
		* 根据id从数据库中获取相关信息
		* on_block()  
			* 将区块生产者和已激活区块生产者求交集
		* schedule_production_loop()
			* start_block()
				* 获取hbs(head block state)
				* 延时等待，避免分叉
				* 设置_pending_block_mode = producing 
				* 若_pending_block_mode = speculating，**返回结果waiting**；
				* 若pending_block_mode = producing，执行：
						* 判断有多少区块需要当前生产者确定
						* chain.abort_block？
						* **chain.start_block()产块**
							* 由当前区块头产生一个新的pending
							* 初始化pending
							* **push_transaction()** 执行交易并将交易回执打包进区块
								* 创建trx_context，初始化，执行action
								* 将回执信息写入trace_receipt
								* 将trx_context中信息写入pending，打包进区块
								* 调用 **emit(accepted_trx, trx)** 发布区块信息
									* 在net_plugin中绑定信号量与插槽
							* 清除过期交易
							* 更新生产者权限
							* 返回到producer_plugin::start_block()
				* 获取pbs(pending_block_state)
				* 调用push_transaction打包为应用的交易和`schedule_trx`
				* 若打包过程中发现下一区块产生时间小于当前时间则**返回结果exhausted**
				* 若出现异常则**返回结果failed**
				* 否则**返回结果succeeded**
			

## 代码解析

### producer_plugin
在producer_plugin.cpp中定义了producer_plugin_impl类，其中定义了如下参数
```C++
boost::program_options::variables_map	_options;	//选项存储器
bool		_production_enabled	= false;	//产块标志
bool		_pause_production	= false;	//暂停产块标志
uint32_t	_production_skip_flags	= 0; 		//eosio::chain::skip_nothing;

using signature_provider_type = std::function<chain::signature_type(chain::digest_type)>;
std::map<chain::public_key_type, signature_provider_type> _signature_providers;           //<公钥，签名提供者>
std::set<chain::account_name>	        _producers;	                //生产者列表
boost::asio::deadline_timer		_timer;		                //定时器，等待一段时间然后执行操作
std::map<chain::account_name, uint32_t>	_producer_watermarks; 	    	//生产者水印，<账户名，32位int - 区块号>
pending_block_mode		        _pending_block_mode;	    	//等待确认模式
transaction_id_with_expiry_index	_persistent_transactions;	//交易id&到期时间 索引表
                                                                                             
int32_t			_max_transaction_time_ms;               //最长交易时间
fc::microseconds	_max_irreversible_block_age_us;         //最大不可逆区块年龄
int32_t		        _produce_time_offset_us = 0;            //
int32_t		        _last_block_time_offset_us = 0;
fc::time_point		_irreversible_block_time;
fc::microseconds	_keosd_provider_timeout_us;       

time_point	        _last_signed_block_time;		//上一签名区块时间
time_point	        _start_time = fc::time_point::now();	//开始时间？
uint32_t		_last_signed_block_num = 0;		//上一签名区块号

producer_plugin* _self = nullptr;

incoming::channels::block::channel_type::handle	            _incoming_block_subscription;	        //频道：incoming_block
incoming::channels::transaction::channel_type::handle	    _incoming_transaction_subscription;		//频道：incoming_trx

compat::channels::transaction_ack::channel_type&            _transaction_ack_channel;	            	//频道：trx_ack

incoming::methods::block_sync::method_type::handle          _incoming_block_sync_provider;          	//方法：同步
incoming::methods::transaction_async::method_type::handle   _incoming_transaction_async_provider;   	//方法：异步

transaction_id_with_expiry_index                            _blacklisted_transactions;              	//黑名单trx

fc::optional<scoped_connection>                             _accepted_block_connection;              	//connection对象
fc::optional<scoped_connection>                             _irreversible_block_connection;
```

### 产块流程1. producer_plugin::plugin_startup()
在producer_plugin启动时开始产块，该函数仅判断生产者列表是否不为空，以及产块标志_production_enabled是否为真，然后调用schedule_production_loop()
```C++
try 
{
	if(fc::get_logger_map().find(logger_name) != fc::get_logger_map().end()) 
	{
		_log = fc::get_logger_map()[logger_name];
	}
	
	//写日志
	ilog("producer plugin:  plugin_startup() begin");
	
	//控制器：chain
	chain::controller& chain = app().get_plugin<chain_plugin>().chain();
	
	//生产者列表不能为空
	EOS_ASSERT( my->_producers.empty() || chain.get_read_mode() == chain::db_read_mode::SPECULATIVE, plugin_config_exception,
              "node cannot have any producer-name configured because block production is impossible when read_mode is not \"speculative\"" );
			  
	//将on_block()绑定到信号量accepted_block上
	my->_accepted_block_connection.emplace(chain.accepted_block.connect( [this]( const auto& bsp ){ my->on_block( bsp ); } ));
	//将on_irreversible_block()绑定在信号量irreversible_block上
	my->_irreversible_block_connection.emplace(chain.irreversible_block.connect( [this]( const auto& bsp ){ my->on_irreversible_block( bsp->block ); } ));
	
	const auto lib_num = chain.last_irreversible_block_num();	//上一不可逆区块号
	const auto lib = chain.fetch_block_by_number(lib_num);		//获取上一区块，返回值为signed_block_ptr
	
	//存在，则设置上一不可逆区块时间
	if (lib) 
	{
		my->on_irreversible_block(lib);				//修改irreversible_block_time
	} else {
		my->_irreversible_block_time = fc::time_point::maximum();		
	}
	
	//生产者列表不为空，开始生产
	if (!my->_producers.empty()) 
	{
		ilog("Launching block production for ${n} producers at ${time}.", ("n", my->_producers.size())("time",fc::time_point::now()));
		
		//产块标记为真
		if (my->_production_enabled) 
		{
			//空链，打印标志
			if (chain.head_block_num() == 0) 
			{
				new_chain_banner(chain);
			}
		}
	}
	
	//启动产块循环
	my->schedule_production_loop();
	
	ilog("producer plugin:  plugin_startup() end");
} FC_CAPTURE_AND_RETHROW() 
```

### 产块流程 2. producer_plugin::schedule_production_loop()
调用start_block()产块

```C++
chain::controller& chain = app().get_plugin<chain_plugin>().chain();
_timer.cancel();				//取消定时器？

//weak_ptr: 与shared_ptr共同工作，用于观测
std::weak_ptr<producer_plugin_impl> weak_this = shared_from_this();

bool last_block;
auto result = start_block(last_block);		//产块
```

### 产块流程 3.producer_plugin::start_block()
* 延时500ms再产块，避免分叉
* 计算下一区块时间，写入`block_time`
* 设置_pending_block_mode为`producing`
* 若满足一下任一条件，将_pending_block_mode设置为`speculating`
	* _production_enabled = false
	* 当前生产者不在scheduled_producer列表
	* 当前生产者签名不在_signature_providers列表
	* _pause_production = true
	* 上一不可逆区块年龄过大(?)
* 若仍处于`producing`状态，则判断是否存在拜占庭行为；若存在，将_pending_block_mode设置为`speculating`
* 若处于`speculating`状态，判断头区块年领是否大于5s；若大于，返回启动结果： `waiting`
* 若仍处于`producing`状态，确定需要当前生产者确认的区块数量，写入`blocks_to_confirm`
* 调用chain.abort_block() (?)
* 调用chain.start_block(block_time, blocks_to_confirm)

```C++
producer_plugin_impl::start_block_result producer_plugin_impl::start_block(bool &last_block) 
{
	//创建控制器chain
	chain::controller& chain = app().get_plugin<chain_plugin>().chain();
	const auto& hbs = chain.head_block_state();    //获取头区块（上一区块）状态
	
	//Schedule for the next second's tick regardless of chain state
	// If we would wait less than 50ms (1/10 of block_interval), wait for the whole block interval.
	//延时，避免分叉
	fc::time_point now = fc::time_point::now();    //当前时间
	fc::time_point base = std::max<fc::time_point>(now, chain.head_block_time()); 
	
	//最小产块间隔， block_interval: 500ms 
	int64_t min_time_to_next_block = (config::block_interval_us) - (base.time_since_epoch().count() % (config::block_interval_us) );
	fc::time_point block_time = base + fc::microseconds(min_time_to_next_block);   //下一区块时间
	
	//若区块间隔<500ms，则再延时500ms
	if((block_time - now) < fc::microseconds(config::block_interval_us/10) ) 
	{     
		// we must sleep for at least 50ms
		block_time += fc::microseconds(config::block_interval_us);
	}
	
	//设置待打包区块状态：producing；开始产块
	_pending_block_mode = pending_block_mode::producing;
	
	// Not our turn
	last_block = ((block_timestamp_type(block_time).slot % config::producer_repetitions) == config::producer_repetitions - 1);
	
	//获取当前时间生产者
	const auto& scheduled_producer = hbs->get_scheduled_producer(block_time);
	auto currrent_watermark_itr = _producer_watermarks.find(scheduled_producer.producer_name);	//<生产者， 区块号>
	auto signature_provider_itr = _signature_providers.find(scheduled_producer.block_signing_key);	//<生产者，签名>
	auto irreversible_block_age = get_irreversible_block_age();	//上一不可逆区块年龄
	
	//在开始产块前，判断是否满足产块条件
	// If the next block production opportunity is in the present or future, we're synced.
	if( !_production_enabled ) 
	{
		_pending_block_mode = pending_block_mode::speculating;	//非生产状态
	}
	else if( _producers.find(scheduled_producer.producer_name) == _producers.end()) 
	{ 
		_pending_block_mode = pending_block_mode::speculating;	//不在生产者列表
	} else if (signature_provider_itr == _signature_providers.end()) 
	{
		elog("Not producing block because I don't have the private key for ${scheduled_key}", ("scheduled_key", scheduled_producer.block_signing_key));
		_pending_block_mode = pending_block_mode::speculating;	//不在签名列表
	} 
	else if ( _pause_production ) 
	{
		elog("Not producing block because production is explicitly paused");
		_pending_block_mode = pending_block_mode::speculating;	//处于暂停生产状态
	} 
	else if ( _max_irreversible_block_age_us.count() >= 0 && irreversible_block_age >= _max_irreversible_block_age_us ) 
	{
		elog("Not producing block because the irreversible block is too old [age:${age}s, max:${max}s]", ("age", irreversible_block_age.count() / 1'000'000)( "max", _max_irreversible_block_age_us.count() / 1'000'000 ));
		_pending_block_mode = pending_block_mode::speculating;	//上一不可逆区块年龄过大(?)
	}
	
	//仍处于处于生产状态，判断是否存在BFT行为
	if (_pending_block_mode == pending_block_mode::producing) 
	{
		// determine if our watermark excludes us from producing at this point
		if (currrent_watermark_itr != _producer_watermarks.end()) 
		{
			//当前区块号 >= 头区块号 + 1，分叉
			if (currrent_watermark_itr->second >= hbs->block_num + 1) 
			{
				elog("Not producing block because \"${producer}\" signed a BFT confirmation OR block at a higher block number (${watermark}) than the current fork's head (${head_block_num})",
                ("producer", scheduled_producer.producer_name)
                ("watermark", currrent_watermark_itr->second)
                ("head_block_num", hbs->block_num));
				_pending_block_mode = pending_block_mode::speculating;
			}
		}
	}
	
	//已经不处于生产状态，若头区块年龄大于5s(？)则返回结果：waiting
	if (_pending_block_mode == pending_block_mode::speculating) 
	{
		auto head_block_age = now - chain.head_block_time();
		if (head_block_age > fc::seconds(5))		//头区块年龄 > 5s
		return start_block_result::waiting;		//返回启动结果： waiting
	}
	
	try
	{
		uint16_t blocks_to_confirm = 0;
		
		//判断该节点有多少区块需要当前生产者确认
		if (_pending_block_mode == pending_block_mode::producing) 
		{
			// determine how many blocks this producer can confirm
			// 1) if it is not a producer from this node, assume no confirmations (we will discard this block anyway)
			// 2) if it is a producer on this node that has never produced, the conservative approach is to assume no
			//    confirmations to make sure we don't double sign after a crash TODO: make these watermarks durable?
			// 3) if it is a producer on this node where this node knows the last block it produced, safely set it -UNLESS-
			// 4) the producer on this node's last watermark is higher (meaning on a different fork)
			if (currrent_watermark_itr != _producer_watermarks.end()) 
			{
				auto watermark = currrent_watermark_itr->second;    //当前区块号
				if (watermark < hbs->block_num) 
				{	//小于头区块号
					//需要确认的区块号
					blocks_to_confirm = std::min<uint16_t>(std::numeric_limits<uint16_t>::max(), (uint16_t)(hbs->block_num - watermark));
				}
			}
		}
		
	chain.abort_block();	//？
	chain.start_block(block_time, blocks_to_confirm);	//产块
} FC_LOG_AND_DROP();
```

### 产块流程4. chain.start_block()
代码中的pending是controller_impl类中的参数，类型为pending_state
结构体pending_state定义如下，可看作是一个待打包的区块
```C++
struct pending_state 
{
	pending_state( database::session&& s )
   :_db_session( move(s) ){}
   
	database::session		_db_session;		//数据库会话
	block_state_ptr			_pending_block_state;	//指针，指向待打包的区块
	vector<action_receipt>	 	_actions;		//区块中包含的action
	controller::block_status	_block_status = controller::block_status::incomplete;		//区块状态默认为: incomplete
	
	void push() { _db_session.push(); }
};
```
具体流程：
* 设置pending参数值
* 判断pending中的区块是否处于pending状态，将结果写入pending
* 判断read_mode是否等于SPECULATIVE，若是则获取全局设置gpo
	* 判断gpo中区块号是否不可逆且存在生产者schedule，且pending中生产者数为0；若是则为pending设置生产者
	* 创建on_block_trx（创建区块的交易），并调用push_transaction将该交易打包进区块
	* 清空过期交易
	* 更新生产者权限

```C++
void start_block( block_timestamp_type when, uint16_t confirm_block_count, controller::block_status s ) 
{
	EOS_ASSERT( !pending, block_validate_exception, "pending block is not available" );
	EOS_ASSERT( db.revision() == head->block_num, database_exception, "db revision is not on par with head block",
                ("db.revision()", db.revision())("controller_head_block", head->block_num)("fork_db_head_block", fork_db.head()->block_num) );
				
	auto guard_pending = fc::make_scoped_exit([this](){pending.reset();});
	
	//pending: 当前待打包区块
	pending = db.start_undo_session(true);
	
	pending->_block_status = s;			//设置状态
	
	//设置block_state：区块头
	pending->_pending_block_state = std::make_shared<block_state>( *head, when ); // promotes pending schedule (if any) to active
	pending->_pending_block_state->in_current_chain = true;
	
	pending->_pending_block_state->set_confirmed(confirm_block_count);			//确认之前区块
	
	//判断是否处于pending状态
	auto was_pending_promoted = pending->_pending_block_state->maybe_promote_pending();
	
	//modify state in speculative block only if we are speculative reads mode (other wise we need clean state for head or irreversible reads)
	//speculative模式：可修改区块状态
	if ( read_mode == db_read_mode::SPECULATIVE || pending->_block_status != controller::block_status::incomplete ) 
	{
		//gpo: 全局设置
		const auto& gpo = db.get<global_property_object>();
		//gpo中区块号不可逆且pending中生产者个数为0
		if( gpo.proposed_schedule_block_num.valid() && // if there is a proposed schedule that was proposed in a block ...
			( *gpo.proposed_schedule_block_num <= pending->_pending_block_state->dpos_irreversible_blocknum ) && // ... that has now become irreversible ...
				pending->_pending_block_state->pending_schedule.producers.size() == 0 && // ... and there is room for a new pending schedule ...
					!was_pending_promoted // ... and not just because it was promoted to active at the start of this block, then:
		)
		{
			// Promote proposed schedule to pending schedule.
			if( !replaying ) 
			{
				ilog( "promoting proposed schedule (set in block ${proposed_num}) to pending; current block: ${n} lib: ${lib} schedule: ${schedule} ",
                        ("proposed_num", *gpo.proposed_schedule_block_num)("n", pending->_pending_block_state->block_num)
                        ("lib", pending->_pending_block_state->dpos_irreversible_blocknum)
                        ("schedule", static_cast<producer_schedule_type>(gpo.proposed_schedule) ) );
			}
			//为pending设置生产者
			pending->_pending_block_state->set_new_producers( gpo.proposed_schedule );
			//清空gpo数据
			db.modify( gpo, [&]( auto& gp ) 
			{
				gp.proposed_schedule_block_num = optional<block_num_type>();
				gp.proposed_schedule.clear();
			});
		}
		
		try 
		{
			//创建on_block_trx
			auto onbtrx = std::make_shared<transaction_metadata>( get_on_block_transaction() );
			auto reset_in_trx_requiring_checks = fc::make_scoped_exit([old_value=in_trx_requiring_checks,this]()
			{
				in_trx_requiring_checks = old_value;
			});
			in_trx_requiring_checks = true;
			//将onbtrx交易打包进区块
			push_transaction( onbtrx, fc::time_point::maximum(), true, self.get_global_properties().configuration.min_transaction_cpu_usage, true );
			} catch( const boost::interprocess::bad_alloc& e  ) {
				elog( "on block transaction failed due to a bad allocation" );
				throw;
			} catch( const fc::exception& e ) {
				wlog( "on block transaction failed, but shouldn't impact block generation, system contract needs update" );
				edump((e.to_detail_string()));
			} catch( ... ) {
			}
			
		clear_expired_input_transactions();			//清除过期交易
		update_producers_authority();				//更新生产者权限
	}
	
	guard_pending.cancel();
}
```

### 产块流程5. chain.push_transaction
在区块中创建一个新的交易入口，对交易信息进行权限校验，同时决定立即/延时执行交易；交易完成后将回执信息等打包进pending；最后调用emit()广播trx信息

参数解释：
* trx: 交易元数据
* trx_context: 交易执行环境，限制资源使用量
* implicit：指示交易/区块是否经过验证

函数具体流程：
* 创建交易跟踪信息`trace`
* 创建交易环境`trx_context`
	* 设置`trx_context`信息
	* 初始化`trx_context`
	* 设置延时信息
* 验证权限
* 执行`trx_context`中交易，并自动更新net，CPU使用量
* 将交易执行后的回执信息写入`trace->receipt`
	* 若交易经过确认(implicit为false)，将trx打包进`pending->_pending_block_state->trxs`
* 将`trx_context`中已经执行的action打包进`pending->_actions`
* 调用emit()函数，将区块信息广播发布
	* 若`trx`尚未被接受，调用emit(self.acceoted_transaction, trx)
	* 调用emit(self.applied_transaction, trx)
* 若`implicit`为false，在未应用交易`unapplied_transaction`中删除trx
* 返回trace

```C++
transaction_trace_ptr push_transaction( const transaction_metadata_ptr& trx,
                                           fc::time_point deadline,
                                           bool implicit,
                                           uint32_t billed_cpu_time_us,
                                           bool explicit_billed_cpu_time = false )
{
	EOS_ASSERT(deadline != fc::time_point(), transaction_exception, "deadline cannot be uninitialized");
	
	transaction_trace_ptr trace;				//交易trace
	try 
	{
		transaction_context trx_context(self, trx->trx, trx->id);      //创建trx_context
		if ((bool)subjective_cpu_leeway && pending->_block_status == controller::block_status::incomplete) 
		{
			trx_context.leeway = *subjective_cpu_leeway;                
		}
		//设置trx_context信息
		trx_context.deadline = deadline;
		trx_context.explicit_billed_cpu_time = explicit_billed_cpu_time;
		trx_context.billed_cpu_time_us = billed_cpu_time_us;
		trace = trx_context.trace;
		try 
		{
			//初始化trx_context
			//implicit = true, 交易未确认
			if( implicit ) 
			{
				trx_context.init_for_implicit_trx();           
				trx_context.can_subjectively_fail = false;
			} 
			else 
			{
				trx_context.init_for_input_trx( trx->packed_trx.get_unprunable_size(),
                                               trx->packed_trx.get_prunable_size(),
                                               trx->trx.signatures.size());
			}
			
			if( trx_context.can_subjectively_fail && pending->_block_status == controller::block_status::incomplete ) 
			{
				check_actor_list( trx_context.bill_to_accounts ); // Assumes bill_to_accounts is the set of actors authorizing the transaction
			}
			
			trx_context.delay = fc::seconds(trx->trx.delay_sec); 
			
			if( !self.skip_auth_check() && !implicit ) 
			{
				//验证权限
				authorization.check_authorization(
                       trx->trx.actions,
                       trx->recover_keys( chain_id ),
                       {},
                       trx_context.delay,
                       [](){}
                       /*std::bind(&transaction_context::add_cpu_usage_and_check_time, &trx_context,
                                 std::placeholders::_1)*/,
                       false
				);
			}
			
			trx_context.exec();		//执行trx_context中的交易			
			trx_context.finalize(); 	//自动更新net和CPU使用量
			
			auto restore = make_block_restore_point();  
			
			//implicit为false，已经过确认
			if (!implicit) 
			{
				//设置回执状态：执行/延时
				transaction_receipt::status_enum s = (trx_context.delay == fc::seconds(0))
                                                    ? transaction_receipt::executed
                                                    : transaction_receipt::delayed;
				//将交易回执信息写入trace->receipt
				trace->receipt = push_receipt(trx->packed_trx, s, trx_context.billed_cpu_time_us, trace->net_usage);
				//将trx打包进pending
				pending->_pending_block_state->trxs.emplace_back(trx);
			} 
			//交易未确认，只设置跟踪信息
			else {
				transaction_receipt_header r;
				r.status = transaction_receipt::executed;
				r.cpu_usage_us = trx_context.billed_cpu_time_us;
				r.net_usage_words = trace->net_usage / 8;
				trace->receipt = r;
			}
			
			//将trx中已经执行的action信息写入pending
			fc::move_append(pending->_actions, move(trx_context.executed));
			
			//call the accept signal but only once for this transaction
			if (!trx->accepted) 
			{
				//广播发布trx信息
				emit( self.accepted_transaction, trx);     //1 接受交易
				trx->accepted = true;
			}
			
			emit(self.applied_transaction, trace);        //2 应用交易
			
			if ( read_mode != db_read_mode::SPECULATIVE && pending->_block_status == controller::block_status::incomplete ) 
			{
				//this may happen automatically in destructor, but I prefere make it more explicit
				trx_context.undo();
			} else {
				restore.cancel();
				trx_context.squash();
			}
			
			//已确认，在未应用交易中删除trx
			if (!implicit) 
			{
				unapplied_transactions.erase( trx->signed_id );
			}
			
			return trace;
			} catch (const fc::exception& e) {
				trace->except = e;
				trace->except_ptr = std::current_exception();
			}
			
			if (!failure_is_subjective(*trace->except)) 
			{
				unapplied_transactions.erase( trx->signed_id );
			}
			
			return trace;
		} FC_CAPTURE_AND_RETHROW((trace))
} /// push_transaction
```
</br>
在chain.start_block()的最后，调用了push_transaction()函数，将交易on_block_trx(onbtrx)打包进pending区块。至此，chain.start_block()函数结束，重新回到producer_plugin::start_block()中。

```C++
push_transaction( onbtrx, fc::time_point::maximum(), true, self.get_global_properties().configuration.min_transaction_cpu_usage, true );
```

### 产块流程6. producer_plugin::start_block()
获取pbs，若pbs为空则直接返回failed；否则首先将未应用的交易中的持久化交易打包进pending区块，再讲剩余交易打包进区块；然后将schedule_trx打包进pending区块。
在此过程中如果发现下一区块产生时间小于当前时间，直接返回结果：exhausted；如果出现异常，直接返回结果：failed。若以上两种情况都不存在，返回结果：succeeded。

详细流程：
* 获取pbs(pending_block_state)，为空则返回结果：failed；否则执行以下过程
* 获取未应用的交易`unapplied_trxs`并删除其中过期的交易
	* 将所有持久化交易打包进pending；若出现异常则返回结果：failed
	* 遍历未应用过的交易
		* 若`block_time`小于当前时间，设置变量`exhuasted`为真，退出
		* 若交易为空，跳过该次循环
		* 交易过期，删除该交易，跳过该次循环
		* 以上情况均不成立，将交易打包进pending；若出现异常则返回结果：failed
* 从blacklist中删除超时交易
* 获取`schedule_trx`并遍历
	* 若`block_time`小于当前时间，设置变量`exhausted`为真，退出
	* 存在_incoming_trx，则调用on_incoming_transaction_async()函数进行异步打包
	* 若交易为空，跳过该次循环
	* 若交易存在于blacklist中，跳过该次循环
	* 以上情况均不成立，将交易打包进pending；若出现异常则返回结果：failed
* 下一区块产生时间`block_time`小于当前时间或者变量`exhausted`为真，返回结果：exhausted
* 否则返回结果：succeed

由于代码过长，在此不再贴代码的详细分析，只具体看一下其中对于`schedule_trx`的处理过程
```C++
//获取schedule_trx
auto scheduled_trxs = chain.get_scheduled_transactions();

//遍历schedule_trx
for (const auto& trx : scheduled_trxs) 
{
	//下一区块产生时间 < 当前时间，设exhausted为真
	if (block_time <= fc::time_point::now()) exhausted = true;
	if (exhausted) { break; }
	
	// configurable ratio of incoming txns vs deferred txns
	//初始化时orig_pending_txn_size = _pending_incoming_transactions.size() 
	//存在_incoming_trx，调用异步打包
	while (_incoming_trx_weight >= 1.0 && orig_pending_txn_size && _pending_incoming_transactions.size()) 
	{
		auto e = _pending_incoming_transactions.front();   
		_pending_incoming_transactions.pop_front();
		--orig_pending_txn_size;     
		_incoming_trx_weight -= 1.0;    
		//异步打包交易
		on_incoming_transaction_async(std::get<0>(e), std::get<1>(e), std::get<2>(e));    
	}
	
	if (block_time <= fc::time_point::now()) 
	{
		exhausted = true;
		break;
	}
	
	//交易存在于blacklist中，跳过该次循环
	if (blacklist_by_id.find(trx) != blacklist_by_id.end()) 
	{
		continue;
	}
	
	try
	{
		auto deadline = fc::time_point::now() + fc::milliseconds(_max_transaction_time_ms);
		bool deadline_is_subjective = false;
		if (_max_transaction_time_ms < 0 || (_pending_block_mode == pending_block_mode::producing && block_time < deadline)) 
		{
			deadline_is_subjective = true;
			deadline = block_time;
		}
		
		//打包交易
		auto trace = chain.push_scheduled_transaction(trx, deadline);
		//存在异常
		if (trace->except) 
		{
			if (failure_is_subjective(*trace->except, deadline_is_subjective))
			{
				exhausted = true; 
			} else
			{
				auto expiration = fc::time_point::now() + fc::seconds(chain.get_global_properties().configuration.deferred_trx_expiration_window);
				
				// this failed our configured maximum transaction time, we don't want to replay it add it to a blacklist
				_blacklisted_transactions.insert(transaction_id_with_expiry{trx, expiration});      //放入黑名单
			}
		}
	} catch ( const guard_exception& e )
	{
		app().get_plugin<chain_plugin>().handle_guard_exception(e);
		return start_block_result::failed;			//存在异常，返回failed
	} FC_LOG_AND_DROP();
	
	_incoming_trx_weight += _incoming_defer_ratio;             //？
	if (!orig_pending_txn_size) _incoming_trx_weight = 0.0;   
}
```

至此producer_plugin::start_block()全部执行结束，接下来要返回到producer_plugin::schedule_production_loop()中。

### 产块流程 7. producer_plugin::schedule_production_loop()
查看start_block()返回的处理结果
若为failed，50ms后重新调用schedule_production_loop()
若返回结果为succeeded或exhausted，并且仍处于生产模式，设置定时器_timer的到期时间，并在到期时间来临时执行maybe_produce_block()进行产块的收尾工作
若处于speculating状态且生产者列表不为空，则准备下一轮的产块

具体实现如下
```C++
auto result = start_block(last_block);       //产块

//失败
if (result == start_block_result::failed) 
{
	elog("Failed to start a pending block, will try again later");
	//block_interval_us: 50ms
	_timer.expires_from_now( boost::posix_time::microseconds( config::block_interval_us  / 10 ));
	
	// we failed to start a block, so try again later?
	_timer.async_wait([weak_this,cid=++_timer_corelation_id](const boost::system::error_code& ec) 
	{
		auto self = weak_this.lock();            //返回shared_ptr
		if (self && ec != boost::asio::error::operation_aborted && cid == self->_timer_corelation_id) 
		{
			self->schedule_production_loop();     //重新调用循环产块过程
		}
	});
} else if (result == start_block_result::waiting) 
{
      // nothing to do until more blocks arrive
} 
//succeeded、exhausted
else if (_pending_block_mode == pending_block_mode::producing) 
{
	// we succeeded but block may be exhausted
	static const boost::posix_time::ptime epoch(boost::gregorian::date(1970, 1, 1));    //创世区块时间？
	if (result == start_block_result::succeeded) 
	{
		// ship this block off no later than its deadline
		_timer.expires_at(epoch + boost::posix_time::microseconds(chain.pending_block_time().time_since_epoch().count() + (last_block ? _last_block_time_offset_us : _produce_time_offset_us)));
		fc_dlog(_log, "Scheduling Block Production on Normal Block #${num} for ${time}", ("num", chain.pending_block_state()->block_num)("time",chain.pending_block_time()));
	}
	
	//exhausted 
	else 
	{
	auto expect_time = chain.pending_block_time() - fc::microseconds(config::block_interval_us);
	
	// ship this block off up to 1 block time earlier or immediately
	//在deadline之前将该区块上链
	if (fc::time_point::now() >= expect_time) 
	{	
		_timer.expires_from_now( boost::posix_time::microseconds( 0 ));     //0s后到期
	} else 
	{
		_timer.expires_at(epoch + boost::posix_time::microseconds(expect_time.time_since_epoch().count()));
	}
	
	fc_dlog(_log, "Scheduling Block Production on Exhausted Block #${num} immediately", ("num", chain.pending_block_state()->block_num));
}

//定时器到期时执行
_timer.async_wait([&chain,weak_this,cid=++_timer_corelation_id](const boost::system::error_code& ec) 
{
	auto self = weak_this.lock();   
	if (self && ec != boost::asio::error::operation_aborted && cid == self->_timer_corelation_id)
	{
		auto res = self->maybe_produce_block();			//收尾产块工作
		fc_dlog(_log, "Producing Block #${num} returned: ${res}", ("num", chain.pending_block_state()->block_num)("res", res) );
	}
});
} 
//非产块状态，且存在生产者
else if (_pending_block_mode == pending_block_mode::speculating && !_producers.empty() && !production_disabled_by_policy())
{
	// if we have any producers then we should at least set a timer for our next available slot
	optional<fc::time_point> wake_up_time;
	for (const auto&p: _producers) 
	{
		auto next_producer_block_time = calculate_next_block_time(p);
		if (next_producer_block_time) 
		{
			//唤醒生产者的时间
			auto producer_wake_up_time = *next_producer_block_time - fc::microseconds(config::block_interval_us);
			if (wake_up_time) 
			{
				// wake up with a full block interval to the deadline
				wake_up_time = std::min<fc::time_point>(*wake_up_time, producer_wake_up_time);
			} else {
				wake_up_time = producer_wake_up_time;
			}
		}
	}
	
	if (wake_up_time)
	{
		fc_dlog(_log, "Specualtive Block Created; Scheduling Speculative/Production Change at ${time}", ("time", wake_up_time));
		static const boost::posix_time::ptime epoch(boost::gregorian::date(1970, 1, 1));
		//设置定时器到期时间
		_timer.expires_at(epoch + boost::posix_time::microseconds(wake_up_time->time_since_epoch().count()));
		//到期后执行
		_timer.async_wait([weak_this,cid=++_timer_corelation_id](const boost::system::error_code& ec) 
		{
			auto self = weak_this.lock();				//返回shared_ptr
			if (self && ec != boost::asio::error::operation_aborted && cid == self->_timer_corelation_id) 
			{
				self->schedule_production_loop();		//产块
			}
		});
	} else {
		fc_dlog(_log, "Speculative Block Created; Not Scheduling Speculative/Production, no local producers had valid wake up times");
	}
} else {
	fc_dlog(_log, "Speculative Block Created");
}
```

### 产块流程8. producer_plugin_impl::maybe_produce_block()
调用producer_block()进行产块的收尾工作
若出现异常则返回false，否则返回true

```C++
bool producer_plugin_impl::maybe_produce_block() 
{
	auto reschedule = fc::make_scoped_exit([this]{
	schedule_production_loop();
	});
	
	try 
	{
		//调用produce_block()进行产块的收尾工作
		produce_block();
		return true;
	} catch ( const guard_exception& e ) 
	{
		app().get_plugin<chain_plugin>().handle_guard_exception(e);
		return false;
	} catch ( boost::interprocess::bad_alloc& ) 
	{
		raise(SIGUSR1);
		return false;
	} FC_LOG_AND_DROP();
	
	fc_dlog(_log, "Aborting block due to produce_block error");
	chain::controller& chain = app().get_plugin<chain_plugin>().chain();
	chain.abort_block();
	return false;
}
```

### 产块流程9. producer_block()
调用chain.finalize_block()为区块添加merkle根
调用chain.sign_block()为区块签名
调用chain.commit_block()提交区块
在日志中记录本次产块信息

具体实现如下
```C++
chain.finalize_block();				//为区块添加merkle根
//签名
chain.sign_block( [&]( const digest_type& d ) 
{  
	auto debug_logger = maybe_make_debug_time_logger();
	return signature_provider_itr->second(d);
} );

	chain.commit_block();			//提交
	auto hbt = chain.head_block_time();
	//idump((fc::time_point::now() - hbt));
	
	block_state_ptr new_bs = chain.head_block_state();
	_producer_watermarks[new_bs->header.producer] = chain.head_block_num();
	
	//在日志中记录信息
	ilog("Produced block ${id}... #${n} @ ${t} signed by ${p} [trxs: ${count}, lib: ${lib}, confirmed: ${confs}]",
		("p",new_bs->header.producer)("id",fc::variant(new_bs->id).as_string().substr(0,16))
		("n",new_bs->block_num)("t",new_bs->header.timestamp)
		("count",new_bs->block->transactions.size())("lib",chain.last_irreversible_block_num())("confs", new_bs->header.confirmed));
```

### 产块流程10. chain.commit_block()
产块流程的倒数第二步，提交区块上链
首先将已经被生产且当前节点确认的区块添加到fork_db中
然后调用emit()广播已经接受区块头的信息
再调用emit()广播已经接受区块的信息
过程中若出错则调用abort_block()撤销产块

具体实现如下
```C++
void commit_block( bool add_to_fork_db ) 
{
	auto reset_pending_on_exit = fc::make_scoped_exit([this]{
         pending.reset();
	});
	
	try
	{
		if (add_to_fork_db) 
		{
			//将已被生产且被当前节点确认的区块添加到fork_db中
			pending->_pending_block_state->validated = true;
			auto new_bsp = fork_db.add(pending->_pending_block_state);
			emit(self.accepted_block_header, pending->_pending_block_state);	//广播 4接受区块头
			head = fork_db.head();
			EOS_ASSERT(new_bsp == head, fork_database_exception, "committed block did not become the new head in fork database");
		}
		
		if( !replaying ) 
		{
			reversible_blocks.create<reversible_block_object>( [&]( auto& ubo ) 
			{
				ubo.blocknum = pending->_pending_block_state->block_num;
				ubo.set_block( pending->_pending_block_state->block );
			});
		}
		
		emit( self.accepted_block, pending->_pending_block_state );		//广播 5接收区块
	} catch (...)
	{
		// dont bother resetting pending, instead abort the block
		reset_pending_on_exit.cancel();
		abort_block();		//出错，撤销产块
		throw;
	}
	
	// push the state for pending.
	pending->push();
}
```

### 产块流程11. emit()
