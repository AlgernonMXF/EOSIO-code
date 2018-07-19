##transaction_metadata.hpp
> 包含交易中上下文无关的缓存数据

```C++
class transaction_metadata
{
public:
	transaction_id_type			id;
	transaction_id_type			signed_id;
	signed_transaction			trx;
	packed_transaction			packed_trx;
}
```
