# transaction
## transaction.h
定义发送交易和内联消息的API
```C++
//发送异步（延期？）交易
1. void send_deferred(const uint128_t& sender_id, account_name payer, const char *serialized_transaction, size_t size, uint32_t replace_existing = 0)
//取消异步交易
2. int cancel_deferred(const uint128_t& sender_id)
//复制当前执行交易到buffer
3. size_t read_transaction(char *buffer, size_t size)
//获得当前执行交易大小
4. size_t transaction_size()
//获得当前执行交易中TAPOS的区块数目
5. int tapos_block_num()
//获得当前执行交易中TAPOS的区块前缀
6. int tapos_block_prefix()
//获取当前执行交易的保质期
7. time expiration()
//从当前活动交易中检索特定操作
8. int get_action( uint32_t type, uint32_t index, char* buff, size_t size )
//从已签署交易中检索上下文无关数据
9. int get_context_free_data( uint32_t index, char* buff, size_t size )
```

## transaction.hpp

> inside namespace eosio

* **class**

```C++
class transaction_header
{
public:
	//构造函数
	transaction_header(time_point_sec exp = time_point_sec(now() + 60)) : expiration(exp) {}
	
	//成员变量
	time_point_sec	expiration;
	uint16_t		ref_block_num;
	uint32_t		ref_block_prefix;
	unsigned_int	net_usage_words = 0UL;		//压缩后本交易可序列化的字节数
	uint8_t			max_cpu_usage_ms = 0UL;		//为本交易计费的CPU使用单元数
	unsigned_int	delay_sec = 0UL;
	
	EOSLIB_SERIALIZE( transaction_header, (expiration)(ref_block_num)(ref_block_prefix)(net_usage_words)(max_cpu_usage_ms)(delay_sec) )

}

class transaction : public transaction_header
{
public:
	//构造函数
	transaction(time_point_sec exp = time_point_sec(now() + 60)) : transaction_header(exp) {}
	
	void send(const uint128_t& sender_id, account_name payer, bool replace_existing = false) const
	
	//成员变量
	vector<action>		context_free_actions;	//上下文无关操作
	vector<action>		actions;				//操作
	extensions_types	transaction_extensions;	
	
	EOSLIB_SERIALIZE_DERIVED( transaction, transaction_header, (context_free_actions)(actions)(transaction_extensions) )
}
```

* **struct**

```C++
//交易出错，卷回
struct onerror
{
	uint128_t	sender_id;		
	bytes		sent_trx;
	
	static from_current_action()
	transaction unpack_sent_trx()
	
	EOSLIB_SERIALIZE( onerror, (sender_id)(sent_trx) )
}
```

* **function**

```C++
//从活动交易中检索特定操作
//type: 0代表content free action, 1代表action
//index: 指定操作的索引
inline action get_action(uint32_t type, uint32_t index)
{
	constexpr size_t max_stack_buffer_size = 512;
	int s = ::get_action( type, index, nullptr, 0 );
	eosio_assert( s > 0, "get_action size failed" );
	size_t size = static_cast<size_t>(s);
	char* buffer = (char*)( max_stack_buffer_size < size ? malloc(size) : alloca(size) );
	auto size2 = ::get_action( type, index, buffer, size );
	eosio_assert( size == static_cast<size_t>(size2), "get_action failed" );
	return eosio::unpack<eosio::action>( buffer, size );
}
```

