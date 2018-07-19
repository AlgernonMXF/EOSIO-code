# transaction.hpp
> 交易，其中包含一系列action

## struct

Line: `36 - 53`
* 交易头：包含于交易相关的固定大小的数据，与交易主体数据分离，便于解析交易。
* 所有交易都有到期时间，过期的区块不会被包含在区块链中；
  一旦区块时间戳大于到期时间，区块会被认定为不可逆，其中包含的交易也永远不会成立。
* 每个region是一个独立的区块链，包含用于通信的路由结点；region内的合约可能会产生或授权region外的交易
```C++
struct transaction_header
{
	time_point_sec		expiration;			//区块到期时间
	uint16_t		ref_block_num		= 0U;	//specifies a block num in the last 2^16 blocks
	uint32_t		ref_block_prefix	= 0UL;	//区块id的低32bit
	fc::unsigned_int	max_net_usage_words 	= 0UL;	//计费网络带宽的上限，单位：8B
	uint8_t			max_cpu_usage_ms	= 0;	//计费CPU时间的上限，单位：ms
	fc::unsigned_int	delay_sec		= 0UL;	//允许交易取消的延迟时间
	
	block_num_tyoe get_ref_blocknum(block_num_type head_blocknum) const;		//获取所在区块号
	void set_reference_block(const block_id_type& reference_block);			//设置区块号
	bool verify_reference_block(const block_id_type& reference_block) const;	//验证区块号
	void validate() const;								//验证
}
```

Line: `62 - `
* 交易由一组必须被全部应用或全部拒绝的消息组成
* 这些消息可访问给定读写范围内的数据
```C++
struct transaction : public transaction_header
{
	vector<action>		context_free_actions;		//上上下文无关action集
	vector<action>		actions;			//action集
	extension_type		transaction_extension;		//
	
	transaction_id_type		id()const;			//交易id
	digest_type			sig_digesty(const chain_id_type&, const vector<bytes>& )	//已签名的摘要？	
	flat_set<public_key_type>	get_signature_keys();		//获取签名钥匙
	uint32_t 			total_actions() const;		//交易中包含的总action数
	account_name			first_authorizor() const;	//首个授权人
}
```

Line: `86 - 104`
已签名交易：包含签名集
```C++
struct signed_transaction : public transaction
{	
	signed_transaction() = default;				//构造函数
	signed_transaction(transaction& , const vector<signature_type>& , const vector<bytes>& cfd) 
	
	vector<signature_type>		signatures;		//签名集
	vector<bytes>			cfd;			//for each context-free action, there is an entry here
	const signature_type&		sign(const private_key_type, const chain_id_ type);		//签名			
	signature_type			sign(const private_key_type, const chain_id_ type) const;	//获取签名
	flat_set<public_key_type>	get_signature_keys(const chian_id_type&, bool allow_duplicate_keys = false) const;	//获取签名钥匙
}	
```

Line: `112 - 159`
```C++
struct packed_transaction
{
	enum conpression_type{none = 0, zlib = 1,};		//压缩类型
	packed_transaction() = default;
	explicit packed_trasaction(const transacrion&, compression_type, _compression = none);
	explicit packed_trasaction(const signed_transacrion&, compression_type, _compression = none);
			
	uint32_t		get_unprunable_size() const;	//不可修改的size？
	uint32_t		get_prunable_size() const;	//可修改的size
	digest_type		packed_digest();		//打包摘要
	
	vector<signature_type>	signatures;			//签名
	fc::enmu_type<uint8_t, compression_type>	compression;//压缩类型
	bytes			packed_context_free_data;	//
	bytes			packed_trx;			//已打包的交易
	
	time_point_sec		expiration()const;		//过期时间
	transaction_id_type	id()const;
	bytes			get_raw_transaction()const;
	vector<bytes>		get_context_free_data()const;
	transaction		get_transaction()const;
	signed_transaction	get_signed_transaction()const;
	void			set_transaction(const transaction& t, compression_type _compression = none);void						set_transaction(const transaction& t, const vector<bytes>& cfd, compression_type _compression = none);
	
private:
	mutable	optional<transaction>	unpacked_trx;
	void			local_unpack()const;	
}
