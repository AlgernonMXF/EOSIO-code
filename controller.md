# contoller - chain控制器

**类中定义的结构体/类如下**
* 区块的pending状态
```C++
struct pending_state 
{
	pending_state( database::session&& s ):_db_session( move(s) ){}
   	database::session		_db_session;			//数据库会话
   	block_state_ptr			_pending_block_state;		//待定区块状态
	vector<action_receipt>		_actions;			//操作收据
	controller::block_status	_block_status = controller::block_status::incomplete;	//区块状态，默认未完成
   	void push() {_db_session.push();}
};
```
* 区块状态
```C++
enum class block_status 
{
	irreversible = 0,	//已被应用，被认为不可逆
	validated   = 1,	//已被有效生产者签名，且被该节点应用过，但尚未不可逆
	complete   = 2,		//已被有效生产者签名，未被应用，尚未不可逆
	incomplete  = 3,	//正被生产者生产或被节点推测性的生产(speculatively produce)？
};
```

## controller类

**类中定义的变量如下表**

|变量名		|类型				|大小	|含义|
|--			|--				|--	|--|
|actor_whitelist	|flat_set<account_name>		|	||
|actor_blacklist	|flat_set<account_name>		|	||
|contract_whitelist	|flat_set<account_name>		|	||
|contract_blacklist	|flat_set<account_name>		|	||
|action_blacklist	|flat_set< pair<account_name, action_name> >|||
|key_blacklist		|flat_set<public_key_type>	|	||
|blocks_dir		|path				|	|区块路径|
|state_dir		|path				|	|状态路径|
|state_size		|uint64_t			|8B	|状态大小|
|reversible_cache_size	|unit64_t			|8B	|可逆缓存大小|
|read_only		|bool				|1B	|是否只读|
|force_all_checks	|bool				|1B	||
|contracts_console	|bool				|1B	||
|genesis		|genesis_state			|	||
|wasm_runtime		|wasm_interface::vm_type	|	|wasm运行时间|


**类中定义的信号量如下表**

|信号量		|类型						|含义|
|----			|----						|----|
|accepted_block_header	|signal<void(const block_state_ptr&)>		|区块头已被接受|
|accepted_block		|signal<void(const block_state_ptr&)>		|区块已被接受|
|irreversible_block	|signal<void(const block_state_ptr&)>		|区块已不可逆|
|accepted_transaction	|signal<void(const transaction_metadata_ptr&)>	|交易已被接受|
|applied_transaction	|signal<void(const transaction_trace_ptr&)>	|交易已被应用|
|accepted_confirmation	|signal<void(const header_confirmation&)>	|确认已被接受|
|bad_alloc		|signal<void(const int&)> 			|坏的分配|                               
               
	       
> 补充说明             
> 此处的signal即signal/slot机制，也被称作观察者模式                    
> signal可通过connect关联多个slot，当signal发出信号，相关的slot都将被调用        
> 详细介绍可参考boost::signal2库

**类中定义的函数如下（只取部分较为重要的解释）**

* 
```C++
void startup();
```

* 开始一个新的pending区块会话（开始产生区块），并将新的trx放入其中
```C++
void start_block( block_timestamp_type time = block_timestamp_type(), uint16_t confirm_block_count = 0 );
void abort_block();		//中止区块
```
* 获取未应用的交易（失败的交易）,丢掉失败的交易
```C++
vector<transaction_metadata_ptr> get_unapplied_transactions() const;
void drop_unapplied_transaction(const transaction_metadata_ptr& trx);
```
* 
```C++
vector<transaction_id_type> get_scheduled_transactions() const;		
transaction_trace_ptr push_scheduled_transaction( const transaction_id_type& scheduled, fc::time_point deadline, uint32_t billed_cpu_time_us = 0 );		//尝试在延迟数据库中执行特定交易？			
```
* 将交易打包进区块/打包区块
```C++
transaction_trace_ptr push_transaction( const transaction_metadata_ptr& trx, fc::time_point deadline, uint32_t billed_cpu_time_us = 0 );
```
* 对区块的操作
```C++
void finalize_block();		//确定区块
void commit_block();		//提交区块
void pop_block();		//弹出区块
void push_block( const signed_block_ptr& b, block_status s = block_status::complete );	//压入区块
```
* 当收到其他节点的确认时调用，可能会更新上一不可逆区块，或引起分叉上的转换
```C++
void push_confirmation( const header_confirmation& c );
```


## controller_impl结构体            
**中定义的变量如下表**

|变量名		|类型			|大小	|含义|
|--			|--			|--	|--|
|self			|controller&		|	||
|db			|chainbase::database	|	|数据库|
|reversible_blocks	|chainbase::database	|	|数据库，存储已应用但仍可逆区块|
|block_log		|blog			|	|区块日志|
|pending		|vector<pending_state>	|	|待定状态集|
|head			|block_state_ptr	|	|区块头指针|
|fork_db		|fork_database		|	|分叉数据库|
|wasmif			|wasm_interface		|	|wasm接口|
|resource_limits	|resource_limits_manager|	|资源管理器|
|authorization		|authrozation_manager	|	|权限管理器|
|conf			|controller::config	|	|权限配置|
|chain_id		|chain_id_type		|	|链id|
|replaying		|bool			|	|，默认为false|
|in_trx_requiring_checks|bool			|	|，默认为false|
|handler_key		|pair<scope_name,action_name> 			|	|处理函数的key|
|apply_handlers		|map<account_name, map<handler_key, apply_handler> >|	|应用处理器？|
|unapplied_transactions	|map<digest_type, transaction_metadata_ptr>	|	|未应用的交易|


**struct中定义的函数如下**
* 区块相关的操作
```C++
void pop_block();			//弹出当前头区块，并将其上一区块设置为当前头区块
void commit_block(bool add_to_fork_db);	//提交区块
```
* 广播区块信息
```C++
void emit( const Signal& s, Arg&& a ) {}
```

### 代码
```C++
//弹出当前头区块，并将其上一区块设置为当前头区块
void pop_block() 
{		
	//获取上一区块
	auto prev = fork_db.get_block( head->header.previous );
      	FC_ASSERT( prev, "attempt to pop beyond last irreversible block" );

	//将当前头区块从reversible_blocks数据库中删除
      	if( const auto* b = reversible_blocks.find<reversible_block_object,by_num>(head->block_num) )
      	{
		reversible_blocks.remove( *b );
      	}
	
	//将当前头区块中的所有交易放入unapplied_transactions，并用交易签名id    
      	for( const auto& t : head->trxs )
         	unapplied_transactions[t->signed_id] = t;
      	head = prev;
      	db.undo();		//？
}

//广播区块信息
void emit( const Signal& s, Arg&& a ) 
{
	try {
        	s(std::forward<Arg>(a));       //转发参数
      	} catch (boost::interprocess::bad_alloc& e) {
         	wlog( "bad alloc" );
         	throw e;
      	} catch ( fc::exception& e ) {
         	wlog( "${details}", ("details", e.to_detail_string()) );
      	} catch ( ... ) {
         	wlog( "signal handler threw exception" );
      	}
}

//
void on_irreversible( const block_state_ptr& s ) 
{
	if( !blog.head() )
        	blog.read_head();

      	const auto& log_head = blog.head();
      	FC_ASSERT( log_head );
      	auto lh_block_num = log_head->block_num();

     	emit( self.irreversible_block, s );
      	db.commit( s->block_num );

      	if( s->block_num <= lh_block_num ) { return; }

      	FC_ASSERT( s->block_num - 1  == lh_block_num, "unlinkable block", ("s->block_num",s->block_num)("lh_block_num", lh_block_num) );
      	FC_ASSERT( s->block->previous == log_head->id(), "irreversible doesn't link to block log head" );
      	blog.append(s->block);

      	const auto& ubi = reversible_blocks.get_index<reversible_block_index,by_num>();
      	auto objitr = ubi.begin();
	while( objitr != ubi.end() && objitr->blocknum <= s->block_num ) 
	{
		reversible_blocks.remove( *objitr );
		objitr = ubi.begin();
     	}
}

//提交区块
void commit_block( bool add_to_fork_db ) 
{
	//在离开时清空
	auto reset_pending_on_exit = fc::make_scoped_exit([this]{ pending.reset(); });

      	try {
		//若添加到fork数据库？(新产生的区块会被添加到fork数据库)
         	if (add_to_fork_db) 
		{
            		pending->_pending_block_state->validated = true;		//设置该区块有效
			
			//将该区块添加到fork数据库，并返回block_state_ptr；添加到数据库头部
            		auto new_bsp = fork_db.add(pending->_pending_block_state);	
			
			//将该区块转发为信号量：已接收的区块头
            		emit(self.accepted_block_header, pending->_pending_block_state);
			
			//判断刚刚提交的区块是否被添加到fork数据库头部
            		head = fork_db.head();
            		FC_ASSERT(new_bsp == head, "committed block did not become the new head in fork database");
         	}

         	if( !replaying ) 
		{
            		reversible_blocks.create<reversible_block_object>( [&]( auto& ubo ) {
               		ubo.blocknum = pending->_pending_block_state->block_num;
               		ubo.set_block( pending->_pending_block_state->block );
            		});
         	}

		//将该区块转发为信号量：已接收的区块
         	emit( self.accepted_block, pending->_pending_block_state );
      	} catch (...) {
         	// dont bother resetting pending, instead abort the block
         	reset_pending_on_exit.cancel();
         	abort_block();
         	throw;
      	}

      	// push the state for pending.
      	pending->push();
}


//将交易收据添加到pending block，并返回自身
template<typename T>                                                                                     
const transaction_receipt& push_receipt( const T& trx, transaction_receipt_header::status_enum status,                            					uint64_t cpu_usage_us, uint64_t net_usage ) 
{                   
	uint64_t net_usage_words = net_usage / 8;                                                             
    	FC_ASSERT( net_usage_words*8 == net_usage, "net_usage is not divisible by 8" );                       
    	pending->_pending_block_state->block->transactions.emplace_back( trx );                               
    	transaction_receipt& r = pending->_pending_block_state->block->transactions.back();                   
    	r.cpu_usage_us         = cpu_usage_us;                                                                
    	r.net_usage_words      = net_usage_words;                                                             
    	r.status               = status;                                                                      
    	return r;                                                                                             
}     

//将交易打包进区块
transaction_trace_ptr push_transaction( const transaction_metadata_ptr& trx,
                                        fc::time_point deadline,
                                        bool implicit,
                                        uint32_t billed_cpu_time_us  )
{
	FC_ASSERT(deadline != fc::time_point(), "deadline cannot be uninitialized");

   	transaction_trace_ptr trace;
   	try {
      		transaction_context trx_context(self, trx->trx, trx->id);
      		trx_context.deadline = deadline;
      		trx_context.billed_cpu_time_us = billed_cpu_time_us;
      		trace = trx_context.trace;
      		try {
         		if( implicit ) {
            		trx_context.init_for_implicit_trx();      //初始化
            		trx_context.can_subjectively_fail = false;
         	} else {
            		trx_context.init_for_input_trx( trx->packed_trx.get_unprunable_size(),
                                            		trx->packed_trx.get_prunable_size(),
                                            		trx->trx.signatures.size() );
         	}

         if( trx_context.can_subjectively_fail && pending->_block_status == controller::block_status::incomplete ) 
	 {
	 	// Assumes bill_to_accounts is the set of actors authorizing the trans
            	check_actor_list( trx_context.bill_to_accounts ); 
         }

         trx_context.delay = fc::seconds(trx->trx.delay_sec);

         if( !self.skip_auth_check() && !implicit ) 
	 {
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

         trx_context.exec();
         trx_context.finalize(); // Automatically rounds up network and CPU usage in trace and bills payers if successful

         auto restore = make_block_restore_point();

         if (!implicit) {
            transaction_receipt::status_enum s = (trx_context.delay == fc::seconds(0))
                                                 ? transaction_receipt::executed
                                                 : transaction_receipt::delayed;
            trace->receipt = push_receipt(trx->packed_trx, s, trx_context.billed_cpu_time_us, trace->net_usage);
            pending->_pending_block_state->trxs.emplace_back(trx);
         } else {
            transaction_receipt_header r;                //交易收据头
            r.status = transaction_receipt::executed;    //执行情况
            r.cpu_usage_us = trx_context.billed_cpu_time_us;
            r.net_usage_words = trace->net_usage / 8;
            trace->receipt = r;                          //将交易回执信息写入回执
         }

         //将交易context中已经执行的action移到区块的pending_action中
         fc::move_append(pending->_actions, move(trx_context.executed));

         //广播区块信息
         // call the accept signal but only once for this transaction
         if (!trx->accepted) {
            //将trx通过完美转发forward变成信号量accepte_transaction
            emit( self.accepted_transaction, trx);
            trx->accepted = true;
         }

         //将trace完美转发成信号量applied_transaction
         emit(self.applied_transaction, trace);

         trx_context.squash();
         restore.cancel();

         if (!implicit) {
            unapplied_transactions.erase( trx->signed_id );
         }
         return trace;
      } catch (const fc::exception& e) {
         trace->except = e;
         trace->except_ptr = std::current_exception();
      }

      if (!failure_is_subjective(*trace->except)) {
         unapplied_transactions.erase( trx->signed_id );
      }

      return trace;
   } FC_CAPTURE_AND_RETHROW((trace))
} /// push_transaction

```
