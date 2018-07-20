# contoller - chain控制器

**类中定义的结构体如下**
* 待确定状态
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
