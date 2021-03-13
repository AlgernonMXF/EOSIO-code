# action.hpp
> 

## struct
Line: `11 - 38`
```C++
//定义权限结构体
struct permission_level
{
	account_name		actor;			//账户名
	permission_level	permission;		//权限
}

//定义权限有关的操作
inline bool operator == (const permission_level&, const permission_level&);
inline bool operator != (const permission_level&, const permission_level&);
inline bool operator < (const permission_level&, const permission_level&);
inline bool operator <= (const permission_level&, const permission_level&);
inline bool operator > (const permission_level&, const permission_level&);
inline bool operator >= (const permission_level&, const permission_level&);
```

Line: '69 - 104'
aciton -> dispatcher -> handles(account scope, function name)

```C++
struct action
{
	account_name			account;			//账户名
	action_name			name;				//操作名
	vector<permission_level>	authorization;			//权限列表
	bytes				data;				//数据
}
```

Line: `106 - 108`
```C++
//action对应的 通知账户
struct action_notice：public action
{
	account_name	receiver;
}
```
