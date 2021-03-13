# block_header_state - 确认交易头的最小必需状态集
**结构体中包含的变量如下：**

|变量名				|类型				|含义|
|--					|--				|--|
|id					|block_id_type			|区块id|
|block_num				|uint32_t			|区块号|
|header					|signed_block_header		|已签名区块头|
|dpos_proposed_irreversible_blocknum	|uint32_t			|DPOS提出的不可逆区块？|
|dpos_irreversible_blocknum		|uint32_t			|DPOS不可逆区块|
|bft_irreversible_blocknum		|uint32_t			|BFT不可逆区块|
|pending_schedule_lib_num		|uint32_t			|上一不可逆区块号|
|pending_schedule_hash			|digest_type			|待定调度hash？|
|pending_schedule			|producer_schedule_type		|待定调度？|
|active_schedule			|producer_schedule_type		|活跃调度|
|blockroot_merkle			|incremental_merkle		|区块根的merkle值？|
|producer_to_last_produced		|flat_map<account_name,uint32_t>|上一区块生产者|
|producer_to_last_implied_irb		|flat_map<account_name,uint32_t>|上一不可逆区块生产者？|
|block_signing_key			|public_key_type		|签署区块的公钥|
|confirm_count				|vector<uint8_t>		||
|confirmations				|vector<header_confirmation>	|确认信息|

**结构体中包含的函数:**

```C++
//给定signed_block_state,生成下一区块头
block_header_state	next( const signed_block_header& h, bool trust = false )const;

//为指定的时间生成一个临时的区块头状态
block_header_state	generate_next( block_timestamp_type when )const;

//设置新的生产者
void 			set_new_producers( producer_schedule_type next_pending );

//将confirmed设置为num_prev_blocks
void 			set_confirmed( uint16_t num_prev_blocks );

//添加confirmation
void 			add_confirmation( const header_confirmation& c );

//获取上一不可逆区块并更新当前区块中相关变量状态
bool 			maybe_promote_pending();

/返回pending_schedule.producer的数量
bool                 	has_pending_producers()const { return pending_schedule.producers.size(); }

//计算上一不可逆区块，需要超过2/3的人同意
uint32_t             	calc_dpos_last_irreversible()const;

//查找上一生产者之外的生产者
bool                 	is_active_producer( account_name n )const;

//获取该区块的生产者
producer_key         	get_scheduled_producer( block_timestamp_type t )const;

//返回上一区块id
const block_id_type& 	prev()const { return header.previous; }

//先求得区块头的签名和区块链根的merkle值的hash值，再与pending_schedule_hash一起求得hash值
digest_type          	sig_digest()const;

//设置生产者签名(producer_signature)
void                 	sign( const std::function<signature_type(const digest_type&)>& signer, bool trust = false );

//返回公钥，谁的？
public_key_type      	signee()const;
};
```
