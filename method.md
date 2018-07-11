# method

> inside namespace appbase
* **using**
	```C++
	using erased_method_ptr = std::unique_ptr<void, void(*)(void*)>
	
	using is_method_decl = decltype(is_method_decl_impl(std::declval<T*>()));
	```

* **struct**
	```C++
	//基本分配策略：将按顺序尝试providers，直到成功完成而不抛出异常
	struct first_succcess_policy;
	
	template<typename Ret, typename ... Args>
	struct first_success_policy<Ret(Args...)>
	{
		using result_type = Ret;
		
		//顺序调用providers，若某个prodiver抛出异常，存储并尝试下一个；若全部失败，则抛出汇总的错误描述
		<template InputIterator>
		Ret operator()(InputIterator first, InputIterator last);
	}
	
	//基本分配策略：仅调用首个provider，并返回结果
	template<typename ... Args>
	struct first_success_policy<void(Args...)> {}
	
	template<typename FunctionSig>
	struct first_provider_policy;
	
	template<typename Ret, typename ... Args>
	struct first_provider_policy<Ret(Args...)> 
	{
      	using result_type = Ret;
		
		template<typename InputIterator>
      	Ret operator()(InputIterator first, InputIterator) 
		{
        	 return *first;
      	}
	};

	template<typename ... Args>
	struct first_provider_policy<void(Args...)> {}
	
	//API特别鉴定器，用于去呗其他方面相同的方法签名
	template< typename Tag, typename FunctionSig, template <typename> class DispatchPolicy = first_success_policy>
   	struct method_decl 
	{
		using method_type = method<FunctionSig, DispatchPolicy<FunctionSig>>;
      	using tag_type = Tag;
	};
	```
	
* **class**
	```C++
	
	//method是松链接的应用级函数，消除了在application中紧耦合不同插件的需要
	//caller可以调用method
	//参数：method签名，分配政策
	template<typename FunctionSig, typename DispatchPolicy>
	class method final : public impl::method_caller<FunctionSig,DispatchPolicy> 
	{
	public:
		class handle
		{
		public:
			~handle() {unregister()};
			void unregister();			//显示取消注册provider
			
			handle() = default;			//构造函数（default？？）
			handle(handle&&) = default;
			handle& operator = (handle&& rhs) = default;
			
			//不允许复制，保护资源
			handle(const handle&) = delete;
			handle& operator = (const handle& ) = delete;	
			
		private:
			using handle_type = boost::signals2::connection;
			handle_type _handle;
			
			//用handle的内部表示构造handle
			handle(handle_type&& _handle) : _handle(std::move(_handle)){}
			
			friend class method;
		}
		
		template<typename T>
		//为method注册一个provider
		handle register_provider(T provider, int priority = 0) 

	protected:
		 method() = default;
		 virtual ~method() = default;
		 
		 static void deleter(void* erased_method_ptr);		//删除
		 static method* get_method(erased_method_ptr& ptr);	//获取method
		 static erased_method_ptr make_unique();			//构建一个独特的指针
		 
		 friend class appbase::application; 
	}
	```
* **function**
	```C++
	template <typename Tag, typename FunctionSig, template <typename> class DispatchPolicy>
   	std::true_type is_method_decl_impl(const method_decl<Tag, FunctionSig, DispatchPolicy>*);

   	std::false_type is_method_decl_impl(...);
	```

> inside namespcae appbase
>> inside namespace impl
	
* **class**
	```C++
	class method_caller
	```
