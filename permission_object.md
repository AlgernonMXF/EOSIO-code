# permission_object - 权限
按层次结构索引，分等级，父权限严格大于子孙权限

## permissino_usage_object
不是很理解，权限使用者？？      
       
代码如下：
```C++
class permission_usage_object : public chainbase::object<permission_usage_object_type, permission_usage_object> 
{
	OBJECT_CTOR(permission_usage_object)
	id_type           id;		
	time_point        last_used;	//< when this permission was last used
};

//索引结构
struct by_account_permission;
using permission_usage_index = chainbase::shared_multi_index_container<
	permission_usage_object,
	indexed_by<
		ordered_unique<tag<by_id>, member<permission_usage_object, permission_usage_object::id_type, &permission_usage_object::id>>
	>
>;
```         


## permission_object
权限           

代码如下：
```C++
class permission_object : public chainbase::object<permission_object_type, permission_object> 
{
	OBJECT_CTOR(permission_object, (auth) )

      	id_type                           id;		//权限id
      	permission_usage_object::id_type  usage_id;	//权限使用id
      	id_type                           parent;	//父权限
      	account_name                      owner; 	//拥有该权限的账户
      	permission_name                   name; 	//权限名，人类可理解
      	time_point                        last_updated; //权限上次更新时间
      	shared_authority                  auth; 	//执行权限所需的authority

       	//判断该权限是否大于或等于other，是则返回真
	template <typename Index>
      	bool satisfies(const permission_object& other, const Index& permission_index) const {
         	// If the owners are not the same, this permission cannot satisfy other
        	if( owner != other.owner )
        		return false;

         	// If this permission matches other, or is the immediate parent of other, then this permission satisfies other
         	if( id == other.id || id == other.parent )
            		return true;

         	// Walk up other's parent tree, seeing if we find this permission. If so, this permission satisfies other
         	const permission_object* parent = &*permission_index.template get<by_id>().find(other.parent);
         	while( parent ) {
            	if( id == parent->parent )
               		return true;
            	if( parent->parent._id == 0 )
               		return false;
            	parent = &*permission_index.template get<by_id>().find(parent->parent);
         	}
	 
         	// This permission is not a parent of other, and so does not satisfy other
         	return false;
      }
      
};

//权限索引，可根据id，parent，owner，name索引
struct by_parent;
struct by_owner;
struct by_name;
using permission_index = chainbase::shared_multi_index_container<
	permission_object,
	indexed_by<
		ordered_unique<tag<by_id>, member<permission_object, permission_object::id_type, &permission_object::id>>,
         	ordered_unique<tag<by_parent>,
			composite_key<permission_object,
				member<permission_object, permission_object::id_type, &permission_object::parent>,
               			member<permission_object, permission_object::id_type, &permission_object::id>
            		>
         	>,
         	ordered_unique<tag<by_owner>,
            		composite_key<permission_object,
               			member<permission_object, account_name, &permission_object::owner>,
               			member<permission_object, permission_name, &permission_object::name>
            		>
         	>,
         	ordered_unique<tag<by_name>,
            		composite_key<permission_object,
               			member<permission_object, permission_name, &permission_object::name>,
               			member<permission_object, permission_object::id_type, &permission_object::id>
            		>
         	>
      	>
>;
```
