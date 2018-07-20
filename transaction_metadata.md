# transaction_metadata - 交易元数据
> 存储交易的上下文无关的缓存数据，包括：打包，解包，压缩和恢复的公钥

## class transaction_metadata
**交易元数据中包含的变量**

|变量名		|类型		|大小	|含义|
|-----		|----			|----	|----|
|id		|transaciton_id_type	|	|交易id	|
|signed_id	|transaciton_id_type	|	|已签名的交易id|
|trx		|signed_transaction	|	|签名的交易？	|
|packed_trx	|packed_transaction	|	|打包的交易？|
|signing_keys	|optional<pair<chain_id_type, flat_set<public_key_type>>>||签名key，包含chain_id|
|accepted	|bool			|	|判断交易是否被认证|

> 补充说明：
> * flat_set      
> flat_set是基于排序vecor的固定量容器；是一个关联查找表，具有o(N)插入时间和o(logN)搜索时间  
> * optional
> c++14中将包含一个std::optional类，它的功能和用法和boost的optional类似。        
> optional<T>内部存储空间可能存储了T类型的值也可能没有存储T类型的值;         
> 只有当optional被T初始化之后，这个optional才是有效的，否则是无效的，它实现了未初始化的概念。        

**交易元数据中包含的函数**       
* 若签名key集未初始化或其中不包含chain_id，则根据chain_id对其初始化
```C++
const flat_set<public_key_type>& recover_keys( const chain_id_type& chain_id )
```
* 获取交易中包含的action数
```C++
uint32_t total_actions()const
```

### 代码
```C++
	
const flat_set<public_key_type>& recover_keys( const chain_id_type& chain_id ) 
{
	// Unlikely for more than one chain_id to be used in one nodeos instance
	//若签名key集未初始化或其中不包含chain_id，则根据chain_id对其初始化
         if( !signing_keys || signing_keys->first != chain_id ) 
            	signing_keys = std::make_pair( chain_id, trx.get_signature_keys( chain_id ) );
         return signing_keys->second;
}

uint32_t total_actions()const 
{
	return trx.context_free_actions.size() + trx.actions.size(); 
}

using transaction_metadata_ptr = std::shared_ptr<transaction_metadata>;
```
