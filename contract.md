# contract

## contract.hpp
所有智能合约的基类

> inside namespace std

```C++
class contract 
{
public:
	//构造函数，account_name代表合约拥有者
	contract( account_name n ):_self(n){}
	//返回合约拥有者
	inline account_name get_self()const{ return _self; }

protected:
	account_name _self;			//合约拥有者
};
```


