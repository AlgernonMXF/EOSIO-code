# transaction.hpp
> 定义交易结构体，其中包含一系列action

## 交易头transaction_header

* 交易头包含于交易相关的固定大小的数据，与交易主体数据分离。
* 避免了解析时的动态内存分配，便于解析交易。
* 所有交易都有到期时间，过期的区块不会被包含在区块链中；
  一旦区块时间戳大于到期时间，区块会被认定为不可逆，其中包含的交易也永远不会成立。
* 每个region是一个独立的区块链，包含用于通信的路由结点；region内的合约可能会产生或授权region外的交易


**交易头中定义的变量如下表：**

|变量（名）		|类型		|大小		|含义|
|----			|----		|----		|----|
|expiration		|time_point_sec	|未知		|交易到期时间|
|ref_block_num		|unit16_t	|2B		|引用的区块号（包含交易的区块号）|	
|ref_block_prefix	|uint32_t	|4B		|引用区块号的低32位|
|max_net_usage_words|unsigned_int	|4B		|计费网络带宽的上限，单位：8B|
|max_cpu_usage_ms	|uint8_t	|1B		|计费CPU时间的上限，单位：ms|
|delay_sec		|unsigned_int	|4B		|允许交易取消的延迟时间|


**交易头中定义的函数：**
* 根据区块号前缀获取区块号:        
```C++
block_num_tyoe get_ref_blocknum(block_num_type head_blocknum) const;
```
* 设置区块号和区块号前缀：        
```C++
void set_reference_block(const block_id_type& reference_block);
```
* 验证区块号：     
```C++
bool verify_reference_block(const block_id_type& reference_block) const;
```
* 验证:        
```C++
void validate() const;
```

### 代码

Line: `36 - 53`
定义交易头结构体
```C++
struct transaction_header
{
	time_point_sec		expiration;			//区块到期时间
	uint16_t		ref_block_num		= 0U;	//specifies a block num in the last 2^16 blocks
	uint32_t		ref_block_prefix	= 0UL;	//区块id的低32bit
	fc::unsigned_int	max_net_usage_words 	= 0UL;	//计费网络带宽的上限，单位：8B
	uint8_t			max_cpu_usage_ms	= 0;	//计费CPU时间的上限，单位：ms
	fc::unsigned_int	delay_sec		= 0UL;	//允许交易取消的延迟时间
	
	block_num_tyoe get_ref_blocknum(block_num_type head_blocknum) const
	{ 
		return ((head_blocknum/0xffff)*0xffff) + head_blocknum%0xffff; 
	}
	
	void set_reference_block(const block_id_type& reference_block);		
	{
		//block_id_type:sha256
		//sha256._hash[4]:int64_t 
		ref_block_num    = fc::endian_reverse_u32(reference_block._hash[0]);
		//问题：_hash[1]是64位，ref_block_prefix是32位???
  	 	ref_block_prefix = reference_block._hash[1];
	}
	bool verify_reference_block(const block_id_type& reference_block) const
	{
		return ref_block_num == (decltype(ref_block_num))fc::endian_reverse_u32(reference_block._hash[0]) &&
			ref_block_prefix == (decltype(ref_block_prefix))reference_block._hash[1];
	}
	void validate() const
	{
		//最大网络带宽使用量不超过2^32
		EOS_ASSERT( max_net_usage_words.value < UINT32_MAX / 8UL, transaction_exception,
               "declared max_net_usage_words overflows when expanded to max net usage" );
	}
}
```
        
	
## 交易 transaction

交易由一组必须被全部应用或全部拒绝的消息组成，这些消息可以访问给定读写范围内的区域

**交易中包含的变量如下表**

|变量名		|类型			|大小	|含义|
|------			|--------		|--	|----|
|context_free_actions	|vector<actiton>|	|上下文无关的action集|
|actions		|vector<action>	|	|相互之间有联系的action集|
|transaction_extensions	|extension_type	|	|	|

**交易中包含的函数如下**

* 获取交易id：         
```C++
transaction_id_type id() const;
```
* 签名摘要：          
```C++
digest_type sig_digest( const chain_id_type& chain_id, const vector<bytes>& cfd = vector<bytes>() ) const;
```
* 从签名中提取公钥：          
```C++
flat_set<public_key_type>  get_signature_keys( const vector<signature_type>& signatures,        
						const chain_id_type& chain_id,        
						const vector<bytes>& cfd = vector<bytes>(),       
						bool allow_duplicate_keys = false ) const;
```
* 获取交易中包含的总操作数;
```C++
uint32_t total_actions()const；
```
* 获取交易的首个授权人？
```C++
account_name first_authorizor()const；
```

### 代码
```C++
struct transaction : public transaction_header
{
	vector<action>		context_free_actions;		//上上下文无关action集
	vector<action>		actions;			//action集
	extension_type		transaction_extension;		//
	
	transaction_id_type id()const
	{
		digest_type::encoder enc;			//digest_type: sha256
		fc::raw::pack( enc, *this );
		return enc.result();				//编码
	}
	
	digest_type sig_digest( const chain_id_type& chain_id, const vector<bytes>& cfd = vector<bytes>() )const
	{
		digest_type::encoder enc;
		fc::raw::pack( enc, chain_id );
		fc::raw::pack( enc, *this );
		if( cfd.size() ) 
		{
			fc::raw::pack( enc, digest_type::hash(cfd) );
   		} 
		else 
		{
      			fc::raw::pack( enc, digest_type() );
   		}
   		return enc.result();                   //编码
	}
	
	//从签名中提取公钥
	flat_set<public_key_type> transaction::get_signature_keys( const vector<signature_type>& signatures, 
									const chain_id_type& chain_id, 
									const vector<bytes>& cfd, 
									bool allow_duplicate_keys )const
	{ 
		try {
   			using boost::adaptors::transformed;

   			constexpr size_t recovery_cache_size = 1000;             //用于恢复的缓冲区大小
   			static recovery_cache_type recovery_cache;               //用于恢复的缓冲区
   			const digest_type digest = sig_digest(chain_id, cfd);    //签名摘要

   			flat_set<public_key_type> recovered_pub_keys;            //恢复的公钥
			//遍历签名
   			for(const signature_type& sig : signatures) 
			{
      				recovery_cache_type::index<by_sig>::type::iterator it = recovery_cache.get<by_sig>().find(sig);
				public_key_type recov;
      				if(it == recovery_cache.get<by_sig>().end() || it->trx_id != id()) {
					//从签名和签名摘要中获取公钥
         				recov = public_key_type(sig, digest);
					//could fail on dup signatures; not a problem
					//将公钥和签名组成的配对放入缓存，以便下次快速访问
        		 		recovery_cache.emplace_back( cached_pub_key{id(), recov, sig} ); 
      				} else {
         				recov = it->pub_key;
      				}
      				bool successful_insertion = false;
      				std::tie(std::ignore, successful_insertion) = recovered_pub_keys.insert(recov);
     				EOS_ASSERT( allow_duplicate_keys || successful_insertion, tx_duplicate_sig,
                  		"transaction includes more than one signature signed using the same key associated with public key: ${key}",
                  		("key", recov)
               			);
   			}		
			
			while(recovery_cache.size() > recovery_cache_size)
      			recovery_cache.erase(recovery_cache.begin());

   			return recovered_pub_keys;
		} FC_CAPTURE_AND_RETHROW() 
	}
	
	//获取交易中包含的总操作数
	uint32_t total_actions()const 
	{ 
		return context_free_actions.size() + actions.size();
	}
	
	//首个授权人？
	account_name first_authorizor()const 
	{               
		for( const auto& a : actions ) 
		{
           	 	for( const auto& u : a.authorization )
              	 	return u.actor;
         	}
         	return account_name();                             //？？函数
      	}
```

## 已签名的交易 signed_transaction

**包含的变量如下表**

|变量（名）		|类型			|大小	|含义|
|---			|---			|---	|---|
|signatures		|vector<signature_type>	|	|签名集|
|context_free_data	|vector<bytes>		|	|上下文无关action的条目|	

**包含的函数：     

* 使用私钥签名，并将签名放入交易的签名集
```C++
const signature_type& sign(const private_key_type& key, const chain_id_type& chain_id);
```
* 使用私钥签名，但不将签名放入交易的签名集
```C++
signature_type sign(const private_key_type& key, const chain_id_type& chain_id)const;
```
* 从签名中提取公钥
```C++
flat_set<public_key_type> get_signature_keys( const chain_id_type& chain_id, bool allow_duplicate_keys = false )const;
```

### 代码
```C++
struct signed_transaction : public transaction
{	
	//构造函数
	signed_transaction() = default;				
	signed_transaction( transaction&& trx, const vector<signature_type>& signatures, const vector<bytes>& 				context_free_data)
      		: transaction(std::move(trx))
      		, signatures(signatures)
      		, context_free_data(context_free_data){}
		
	//签名
	const signature_type& sign(const private_key_type& key, const chain_id_type& chain_id)
	{	
		//使用私钥签名，并将签名放入签名集
		signatures.push_back(key.sign(sig_digest(chain_id, context_free_data)));
  		return signatures.back();
	}		
	signature_type sign(const private_key_type& key, const chain_id_type& chain_id) const
	{
		//签名，但不将签名放入交易的签名集
		return key.sign(sig_digest(chain_id, context_free_data));
	}
	
	//从签名中提取公钥
	flat_set<public_key_type>	get_signature_keys(const chian_id_type&, bool allow_duplicate_keys = false) const
	{	
		return transaction::get_signature_keys(signatures, chain_id, context_free_data, allow_duplicate_keys);
	}
}	
```

## 打包的交易 packed_transaction            

**包含的变量如下表**

|变量（名）		|类型					|大小	|含义|
|----			|----					|----	|----|
|signatures		|vector<signature_type>			|	|签名集|
|conpression		|fc::enum_type<uint8_t,compression_type>|	|压缩类型|
|packed_context_free_data|bytes					|	|	|
|packed_trx		|bytes					|	|已打包的交易|   
|unpacked_trx		|mutable optional<transaction>		|	|用于检索的中间缓冲区|

**包含的函数如下**
* 可修改和不可修改的size
```C++
uint32_t get_unprunable_size()const;
uint32_t get_prunable_size()const;
```
* 打包摘要
```C++
digest_type packed_digest()const;
```
* 获取交易到期时间
```C++
time_point_sec expiration()const; 
```
* 获取交易id
```C++
transaction_id_type id()const;
```
* 返回解包交易
```C++
bytes get_raw_transaction()const;
```
* 解包，返回cfd
```C++
vector<bytes> get_context_free_data()const;
```
* 解包，返回trx
```C++
transaction get_transaction()const;
```
* 返回signed_trx
```C++
signed_transaction get_signed_transaction()const;
```
* 根据压缩类型设置交易
```C++
void set_transaction(const transaction& t, compression_type _compression = none);
void set_transaction(const transaction& t, const vector<bytes>& cfd, compression_type _compression = none);
```

### 代码
```C++
struct packed_transaction
{
	enum conpression_type{none = 0, zlib = 1,};		//压缩类型
	//构造函数
	packed_transaction() = default;
	explicit packed_trasaction(const transacrion&, compression_type, _compression = none);
	explicit packed_trasaction(const signed_transacrion&, compression_type, _compression = none);
			
	uint32_t get_unprunable_size() const
	{
		uint64_t size = config::fixed_net_overhead_of_packed_trx;
   		size += packed_trx.size();		//已打包的交易
  		FC_ASSERT( size <= std::numeric_limits<uint32_t>::max(), "packed_transaction is too big" );
   		return static_cast<uint32_t>(size);
	}
	
	uint32_t get_prunable_size() const
	{
		uint64_t size = fc::raw::pack_size(signatures);
   		size += packed_context_free_data.size();	//已打包的cfd
   		FC_ASSERT( size <= std::numeric_limits<uint32_t>::max(), "packed_transaction is too big" );
   		return static_cast<uint32_t>(size);
	}
	
	digest_type packed_digest()
	{
		digest_type::encoder prunable;
   		fc::raw::pack( prunable, signatures );			//打包签名集
  	 	fc::raw::pack( prunable, packed_context_free_data );	//打包cfd

  		 digest_type::encoder enc;
   		fc::raw::pack( enc, compression );			//打包压缩类型
   		fc::raw::pack( enc, packed_trx  );			//打包交易
   		fc::raw::pack( enc, prunable.result() );		

   		return enc.result();
	}
	
	time_point_sec expiration()const
	{	
		local_unpack();				//解包
   		return unpacked_trx->expiration;	//返回到期时间
	}
	transaction_id_type id()const
	{
		local_unpack();				//解包
   		return get_transaction().id();		//返回交易id
	}
	
	bytes get_raw_transaction()const
	{
		//解包，返回解包后的数据
		try {
     	 		switch(compression) {
         			case none:
            				return packed_trx;
		 		case zlib:
            				return zlib_decompress(packed_trx);
         			default:
			 		FC_THROW("Unknown transaction compression algorithm");
      			}
   		} FC_CAPTURE_AND_RETHROW((compression)(packed_trx))
	}
	
	vector<bytes> get_context_free_data()const
	{
		//解包，并返回cfd
		try {
      			switch(compression) {
         			case none:
            				return unpack_context_free_data(packed_context_free_data);
         			case zlib:
            				return zlib_decompress_context_free_data(packed_context_free_data);
         			default:
            				FC_THROW("Unknown transaction compression algorithm");
      			}
   		} FC_CAPTURE_AND_RETHROW((compression)(packed_context_free_data))
	}
	
	//获取解包的交易
	transaction get_transaction()const
	{	
		local_unpack();
  		return transaction(*unpacked_trx);
	}
	
	//获取签名的交易
	signed_transaction get_signed_transaction()const
	{	
		try {
      			switch(compression) 
			{
         			case none:
           				 return signed_transaction(get_transaction(), signatures, unpack_context_free_data(packed_context_free_data));
         			case zlib:
            				return signed_transaction(get_transaction(), signatures, 		zlib_decompress_context_free_data(packed_context_free_data));
         			default:
            				FC_THROW("Unknown transaction compression algorithm");
     		 	}
   		} FC_CAPTURE_AND_RETHROW((compression)(packed_trx)(packed_context_free_data))
	}
	
	//根据压缩类型设置交易
	void set_transaction(const transaction& t, compression_type _compression = none)
	{
		try {
      			switch(_compression) {
         			case none:
            				packed_trx = pack_transaction(t);
           	 			break;
         			case zlib:
            				packed_trx = zlib_compress_transaction(t);
            				break;
         			default:
            				FC_THROW("Unknown transaction compression algorithm");
      			}
   		} FC_CAPTURE_AND_RETHROW((_compression)(t))
   		packed_context_free_data.clear();
   		compression = _compression;
	}
	
	//根据压缩类型设置交易和cfd
	void set_transaction(const transaction& t, const vector<bytes>& cfd, compression_type _compression = none)
	{
		 try {
      			switch(_compression) {
         			case none:
            				packed_trx = pack_transaction(t);
            				packed_context_free_data = pack_context_free_data(cfd);
            				break;
         			case zlib:
            				packed_trx = zlib_compress_transaction(t);
            				packed_context_free_data = zlib_compress_context_free_data(cfd);
            				break;
         			default:
            				FC_THROW("Unknown transaction compression algorithm");
      			}
   		} FC_CAPTURE_AND_RETHROW((_compression)(t))
   		compression = _compression;
	}
	
	//本地解包
	void local_unpack()const
	{
		 try {
      			switch(_compression) {
         			case none:
           	 			packed_trx = pack_transaction(t);
            				packed_context_free_data = pack_context_free_data(cfd);
            				break;
         			case zlib:
            				packed_trx = zlib_compress_transaction(t);
            				packed_context_free_data = zlib_compress_context_free_data(cfd);
            				break;
         			default:
            				FC_THROW("Unknown transaction compression algorithm");
      			}
   		} FC_CAPTURE_AND_RETHROW((_compression)(t))
  		compression = _compression;
	}
}
