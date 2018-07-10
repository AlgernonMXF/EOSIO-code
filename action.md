## action - EOSIO操作

### action.h
> 定义用于查询action`属性`的API
可供合约查询当前action并检查是否正确执行

```C++
//拷贝当前action的数据到指定位置，返回拷贝的长度
1. uint32_t read_action_data(void *msg, uint32_t len)
//获取当前action数据字段的长度
2. uint32_t action_data_size()
//将指定账户添加到要通知的账户集合
3. void require_recipient(account_name)
//检测account_name的权限是否在action的权限列表中，不在则抛出异常
4. void require_auth(account_name)
//同上，返回bool值
5. bool has_auth(account_name)
//同上，附加权限名
6. void require_auth(account_name, permission_name)
//判断账户是否存在
7. bool is_account(account_name)
//发送一个内联action到该action父辈交易的上下文(?)
8. void send_inline(char* serialized_action, size_t size)
//发送一个内联，但上下文无关的action
9. void send_context_free_inline(char* serialized_action, size_t size)
//检测给定account_name在当前action上是否村子啊读取锁
10. void require_write_lock(account_name)
//检测当前account_name在当前action上是否存在写入锁
11. void require_read_lock(account_name)
//获取 当前action的发布时间-1970
12. uint64_t publication_time()
//获取当前action的接受者
13. account_name current_receiver()
```

### action.hpp
Type-safe C++ wrapers for Action C API

> inside namespace eosio
	
* **struct**
 
	```C++
	//权限等级
	struct pemission_level
	{
		account_name actor;
		permission_name permission;
	
		friend bool operator==(const permission_level&, permission_level&)
	
		EOSLIB_SERIALIZE(permission-level, (actor)(permission))
	}
	
	struct action
	{
		//action所在合约账户名
		account_name			account;
		action_name			name;
		//权限列表
		vector<permission_level>	authorization;
		//action数据，bytes是std::vector<char>别名
		bytes				date;
		
		//template<typename Action>
		action(vector<permission_level>&& auth, const Action& value>)
		action(const permission_level& auth, const Action& value):authorization(1, auth)
		action(const Action&)
		
		action(const permission_level& auth, account_name a, action_name n, T&& value)
			:account(a), name(n), authorization(1, auth), data(pack(std::forward<T>(value))) {}

		action(vector<permission_level> auths, account_name a, action_name n, T&& value)
			:account(a), name(n), authorization(std::move(auths)), data(pack(std::forward<T>(value))) {}

		EOSLIB_SERIALIZE( action, (account)(name)(authorization)(data) )

		//内部调用send_inline
		void send() const
		//内部调用send_context_free_inline
		void send_context_free()const
		//将数据解包成输入数据流
		T data_as()
	}
	
	struct action_meta
	{
		static uint64_t get_account() { return Account; }
		static uint64_t get_name()  { return Name; }
	}
	
	//action处理器inline_dispatcher
	struct inline_dispatcher;
	
	//template<typename T, uint64_t Name, typename... Args>
	struct inline_dispatcher<void(T::*)(Args...), Name>
	{
		static void call(account_name, const permission_level&, std::tuple<Args...> args)
		static void call(account_name, vector<permission_level>, std::tuple<Args...> args)
	}
	```
* **function**

	```C++
	//模板函数：解包；调用datastream.hpp中的unpack()
	1. T unpack_action_data()
	//将accounts加入到将被通知的账户集合
	2. void require_recipient(account_name, ...)
	//检测账户权限
	3. void require_auth(account_name, ...)
	//处理action
	4. void dispatcher_inline(account_name, action_name, vector<permission_level>, std::tuple<Args...> args)
	```



