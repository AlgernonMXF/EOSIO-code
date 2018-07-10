# dispatcher.hpp
根据传入的请求，分发执行action

* **函数**

```C++
//code代表action所在合约的账户名，act为action名字
//验证后，调用unpack_action_data()，解包action携带数据
template<typename Contract, typename FirstAction>
bool dispatch(uint64_t code, uint64_t act)

//执行action
template<typename T, typename Q, typename... Args>
bool execute_action( T* obj, void (Q::*func)(Args...)  ) 
```

* **define**

```C++
//内部调用execute_action()，执行一个action
define EOSIO_API_CALL(r, op ,elem)
 
//内部循环调用EOSIO_API_CALL
define EOSIO_API(TYPE, MEMBERS)

//用于编写智能合约
define EOSIO_ABI(TYOE, MEMBERS)
```
