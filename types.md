## types - EOSIO中自定义类型和函数

### types.h

* **typedef**

	```C++
	uint64_t	:	account_name,		//账户名
             		permission_name,	//权限名 
				table_name,		//数据表名
				scope_name,		//域名
				action_name;		//操作名
	uint32_t	:	time;			//时间
	uint16_t	:	weight_type;		//权重类型
	```

* **define**
	```C++
	//X按16字节对齐
	ALIGNED(X) _attribute_((aligned(16))) x
	```


* **typedef struct**
	```C++
	checksum256   :	transaction_id_type	//事务ID类型
	checksum256   :	block_id_type		//区块ID类型
	```

* **struct**
	```C++
	//公钥
	public_key		:	char data[34]
	//签名
	signature		:	uint8_t data[66]aligned
	//sha-256校验码
	ALIGNED(CHECKSUM256)	:	uint8_t hash[32]
	//sha-160校验码
	ALIGNED(CHECKSUM160)	:	uint8_t hash[20]
	//sha-512校验码
	ALIGNED(CHECKSUM512)	:	uint8_t hash[64]
	```

### types.hpp
> #### inside namespace eosio

* **typedef**
	
	```C++
	//在交易transaction中使用
	vector<tuple<uint16_t, vector<char>>>	:	extensions_type
	```
* **define**
	```C++
	N(X) ::eosio::string_to_name(#X)
	```
* **struct**
	```C++
	//将uint64_t封装为name，确保只传递给name类型的方法
	//无数学计算
	//已封装输出方法，确保输出base32字符串
	struct name
	{
		account_name value = 0;
		
		//重载隐式类型转换，uint64_t类型会转换为account_name类型
		operaotr uint64_t() const;
		//将name值转换为string
		std::string to_string() const;
		//重载==运算符
		friend bool operator==(const name&, const name&)
	
	private:
		//去掉name值中最右边的‘.‘
		static void trim_right_dos(std::string&)
	}		
	```
* **function**
	注：name中允许包含的字符: `12345abcdefghijklmnopqrstuvwxyz`
	```C++
	//将base32类型转换成数字
	static constexpr char char_to_symbol(char)
	//将char*转换成uint64_t
	static constexpr uin64_t string_to_name(const char*)
	//获取name后缀
	static constexpr uint64_t name_suffix(uint64_t)
	```

> #### inside namespace std

* **struct**
	
	```C++
	//重载运算符()，比较两个sha-256校验码的大小
	struct less<checksum256> : binary_function<checksum256, checksum256, bool>
	{
		bool operator()(const checksum256&, const checksum256&) const
	}
	```
	
> #### inside global namespace

```C++
//重载运算符==, !=
bool operator==(const checksum256&, const checksum256&)
bool operator==(const checksum160&, const checksum160&)
bool operator!=(const checksum160&, const checksum160&)
```

