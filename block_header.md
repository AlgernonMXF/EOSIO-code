# bolck_header区块头
**结构体中包含的变量如下：**

|变量名			|类型		|含义|
|--			|--			|--|
|timestamp		|block_timestampe_type	|block产生的时间|
|producer		|account_name		|生产该区块的账户名|
|confirmed		|uint16_t		|标记是否已经被确认？默认值为1|
|previous		|block_id		|上一区块|
|transaction_mroot	|checksum256_type	|mroot of cycles_summary|
|action_mroot		|checksum256_type	|mrrot of all delivered action receipts（区块中所含所有action收据的merkle根）|
|schedule_version	|uint32_t		|生产者调度器版本？默认值为0|
|new_producers		|optional<producer_schedule_type>|新的生产者，可影响该区块？|
|header_extensions	|extension_type		|扩展|

**结构体中包含的函数如下：**
```C++
//摘要
digest_type	digest()const;		

//获取区块id
block_id_type	id()const;	

//从上一区块id获取区块num
uint32_t	block_num()const {return num_from_id(previous) + 1;}

//从区块id获取区块号
static uint32_t	num_from_id(const block_id_type& id);
```
  
## 已签名的区块头signed_block_header
**继承区块头，新增加变量如下:**

|变量名		|类型		|含义|
|--			|--		|--|
|producer_signature	|signature_type	|生产者签名|

## 区块认证信息header_confirmation
**结构体中定义的变量如下:**

|变量名		|类型	|含义|
|--			|--		|--|
|block_id		|block_id_type	|区块id|
|producer		|account_name	|区块生产者|
|producer_signature	|signature_type	|生产者签名|

