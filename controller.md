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

**类中定义的函数如下**

void startup();

void start_block( block_timestamp_type time = block_timestamp_type(), uint16_t confirm_block_count = 0 );
void abort_block();

vector<transaction_metadata_ptr> get_unapplied_transactions() const;
void drop_unapplied_transaction(const transaction_metadata_ptr& trx);
vector<transaction_id_type> get_scheduled_transactions() const;
transaction_trace_ptr push_transaction( const transaction_metadata_ptr& trx, fc::time_point deadline, uint32_t billed_cpu_time_us = 0 );
transaction_trace_ptr push_scheduled_transaction( const transaction_id_type& scheduled, fc::time_point deadline, uint32_t billed_cpu_time_us = 0 );
void finalize_block();
void commit_block();
void pop_block();
void push_block( const signed_block_ptr& b, block_status s = block_status::complete );



## controller_impl类            
**类中定义的变量如下表**

|变量名		|类型			|大小	|含义|
|--			|--			|--	|--|
|self			|controller&		|	||
|db			|chainbase::database	|	|数据库|
|reversible_blocks	|chainbase::database	|	|数据库，存储已应用但仍可逆区块|
|block_log		|blog			|	|区块日志|
|pending		|vector<pending_state>	|	|待定状态集|
|head			|block_state_ptr	|	|区块状态头|
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

**类中定义的函数如下**
* 弹出区块
```C++
void pop_block()；		
```

### 代码
```C++
//弹出区块，该区块必须可逆
void pop_block() 
{		
	//上一区块
	auto prev = fork_db.get_block( head->header.previous );
      	FC_ASSERT( prev, "attempt to pop beyond last irreversible block" );

	//弹出后同时在reversible_blocks数据库中删除
      	if( const auto* b = reversible_blocks.find<reversible_block_object,by_num>(head->block_num) )
      	{
		reversible_blocks.remove( *b );
      	}
	
      	for( const auto& t : head->trxs )
         	unapplied_transactions[t->signed_id] = t;
      	head = prev;
      	db.undo();		//撤销（上次操作？
}

//设置应用处理函数                                                            
void set_apply_handler( account_name receiver, account_name contract, action_name action, apply_handler v ) 
{
	apply_handlers[receiver][make_pair(contract,action)] = v;
}

//转发?
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

```
