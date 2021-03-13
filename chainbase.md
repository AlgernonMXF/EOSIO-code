# Chainbase

> 可参考官方文档说明
> 地址：https://github.com/EOSIO/chainbase

## 功能 
* 可快速控制的事物数据库
* 记录所有撤销记录
* 状态在多个过程中时持久且可共享的
* 嵌套事务具有撤销更改的能力
* 旨在满足区块链应用程序的要求
* 适用于所有需要保持复杂应用程序状态并局有撤销能力的程序

## 代码分析

### 定义锁相关：(line: 31 - 41)
* 锁数量
* 锁请求
  
```C++
#ifndef CHAINBASE_NUM_RW_LOCKS
   #define CHAINBASE_NUM_RW_LOCKS 10                    //读写锁数量:10个
#endif

#ifdef CHAINBASE_CHECK_LOCKING
   #define CHAINBASE_REQUIRE_READ_LOCK(m, t) require_read_lock(m, typeid(t).name())
   #define CHAINBASE_REQUIRE_WRITE_LOCK(m, t) require_write_lock(m, typeid(t).name())
#else
   #define CHAINBASE_REQUIRE_READ_LOCK(m, t)            //请求共享锁
   #define CHAINBASE_REQUIRE_WRITE_LOCK(m, t)           //请求互斥锁
#endif
```

### 定义命名空间chainbase中的变量、函数、数据结构：（line: 47 - ）
#### 定义命名空间中的全局变量和函数:（line: 49 - 96 ）

```C++
   namespace bip = boost::interprocess;                 //进程间通信，包括共享内存，内存映射文件，同步等
   namespace bfs = boost::filesystem;                   //文件系统
   using std::unique_ptr;                               //将std::unique_ptr添加到chainbase的声明区域中
   using std::vector;
   
   /**
    * managed_mapped_file：托管内存映射文件
    * 允许在共享内存和内存映射文件上构建复杂的boost数据结构
    *
    * allocator:在c++的STL中，作为容器标配，负责所有数据结构的动态内存的分配、销毁，对象的构造和析构
    * 功能近似new
    */
   template<typename T>
   using allocator = bip::allocator<T, bip::managed_mapped_file::segment_manager>;
   
   //定义shared_string
   typedef bip::basic_string< char, std::char_traits< char >, allocator< char > > shared_string;

   template<typename T>
   using shared_vector = std::vector<T, allocator<T> >;

   constexpr char _db_dirty_flag_string[] = "db_dirty_flag";

   typedef boost::interprocess::interprocess_sharable_mutex read_write_mutex;       //信号量mutex
   typedef boost::interprocess::sharable_lock< read_write_mutex > read_lock;        //共享锁
   typedef boost::unique_lock< read_write_mutex > write_lock;                       //互斥锁
```

#### 定义chainbase命名空间中的类：
* oid: object ID type
* undo_state：撤销状态
* int_incrementer：
* generic_index：通用索引（用于管理）MultiIndexType
* abstract_session：抽象会话类
* session_impl：会话实现类
* abstract_index：抽象索引类
* index_impl：索引实现类
* index：索引类
* read_write_mutex_manager：信号量管理
* database：数据库
```C++
   //cbject ID type：对象ID类型
   template<typename T>
   class oid {
      public:
         oid( int64_t i = 0 ):_id(i){}

         oid& operator++() { ++_id; return *this; }

         friend bool operator < ( const oid& a, const oid& b ) { return a._id < b._id; }
         friend bool operator > ( const oid& a, const oid& b ) { return a._id > b._id; }
         friend bool operator == ( const oid& a, const oid& b ) { return a._id == b._id; }
         friend bool operator != ( const oid& a, const oid& b ) { return a._id != b._id; }
         friend std::ostream& operator<<(std::ostream& s, const oid& id) {
            s << boost::core::demangle(typeid(oid<T>).name()) << '(' << id._id << ')'; return s;
         }

         int64_t _id = 0;
   };
   
   //撤销状态
   template< typename value_type >
   class undo_state
   {
      public:
         typedef typename value_type::id_type                      id_type;
         typedef allocator< std::pair<const id_type, value_type> > id_value_allocator_type;
         typedef allocator< id_type >                              id_allocator_type;

         template<typename T>
         undo_state( allocator<T> al )
         :old_values( id_value_allocator_type( al.get_segment_manager() ) ),
          removed_values( id_value_allocator_type( al.get_segment_manager() ) ),
          new_ids( id_allocator_type( al.get_segment_manager() ) ){}

         typedef boost::interprocess::map< id_type, value_type, std::less<id_type>, id_value_allocator_type >  id_value_type_map;
         typedef boost::interprocess::set< id_type, std::less<id_type>, id_allocator_type >                    id_type_set;

         id_value_type_map            old_values;                   //旧值
         id_value_type_map            removed_values;               //要被修改的值？
         id_type_set                  new_ids;                      //新id
         id_type                      old_next_id = 0;
         int64_t                      revision = 0;					//修订号
   };
   
   /**
    * try{
    * }finally ：异常发生时清除资源
    * */
   //手动实现finally
   class int_incrementer
   {
      public:
         int_incrementer( int32_t& target ) : _target(target){ ++_target; }
         ~int_incrementer(){ --_target; }				//析构时恢复
         int32_t get()const{ return _target; }

      private:
         int32_t& _target;
   };
   
   
   //通用索引
   //MultiIndexType至少含有int型id，用作主键；被generic_index赋值和管理
   template<typename MultiIndexType>
   class generic_index
   {
      public:
         typedef bip::managed_mapped_file::segment_manager             segment_manager_type;		//段内存管理器
         typedef MultiIndexType                                        index_type;					//多种索引（区分多级索引）
         typedef typename index_type::value_type                       value_type;					//值类型
         typedef bip::allocator< generic_index, segment_manager_type > allocator_type;				
         typedef undo_state< value_type >                              undo_state_type;				//撤销状态类型

		 //构造函数
         generic_index( allocator<value_type> a )
         :_stack(a),_indices( a ),_size_of_value_type( sizeof(typename MultiIndexType::node_type) ),_size_of_this(sizeof(*this)){}

         void validate()const {}											//验证数据类型

         //构造一个新元素，并将id设为下一可用ID，同时增加_next_id，并触发on_create()
         template<typename Constructor>
         const value_type& emplace( Constructor&& c ) {}
		 
         template<typename Modifier>
         void modify( const value_type& obj, Modifier&& m ) {}				//修改元素值
         void remove( const value_type& obj ) {}							//删除元素

         template<typename CompatibleKey>
         const value_type* find( CompatibleKey&& key )const {}				//查找匹配的钥匙？
         const value_type& get( CompatibleKey&& key )const {}				//获取匹配的钥匙

         const index_type& indices()const { return _indices; }

         //会话， //每次会话即一次修订
         class session {
            public:
               session( session&& mv ):_index(mv._index),_apply(mv._apply){ mv._apply = false; }	
               ~session() {if( _apply ) {_index.undo();}}						//析构函数

               void push()   { _apply = false; }								//未应用，则将状态放在栈底
               void squash() { if( _apply ) _index.squash(); _apply = false; }	//将当前会话与上次会话组合
               void undo()   { if( _apply ) _index.undo();  _apply = false; }	//撤销会话
               session& operator = ( session&& mv ) {}							//赋值
               int64_t revision()const { return _revision; }					//返回当前修订号

            private:
               friend class generic_index;	
               session( generic_index& idx, int64_t revision ):_index(idx),_revision(revision) {}      
               generic_index& _index;						//通用索引
               bool           _apply = true;				//当前会话是否被应用
               int64_t        _revision = 0;				//修订号
         };
		 
         session start_undo_session( bool enabled ) {}		//启动撤销会话？
         int64_t revision()const { return _revision; }
		 
         void undo() {}										//将状态恢复到当前会话之前的状态
         void squash(){}									//将最近两次修订合并为一个，不修改索引状态，只更改undo缓冲区的状态；降低修订号    
         void commit( int64_t revision ){}					//舍弃所有修订并提交
         void undo_all(){while( enabled() )undo();}			//展开所有撤销状态？？撤销所有修改
         void set_revision( uint64_t revision ){}			//设置修订版本
         void undo_all()
         void remove_object( int64_t id ){}					//删除（索引）对象

      private:
         bool enabled()const { return _stack.size(); }      //enabled: 返回栈大小？？
         void on_modify( const value_type& v ) {}			//修改栈底（最新）索引数据
         void on_remove( const value_type& v ) {}			//删除栈底数据
         void on_create( const value_type& v ) {}			//在栈底插入新数据

         /**
          * deque：双向队列，支持两端元素的恒定插入和删除
          * 支持随机存储和删除
          * 使用段式内存管理方案，比vector节省内存
          */
         boost::interprocess::deque< undo_state_type, allocator<undo_state_type> > _stack;
		
         int64_t                         _revision = 0;				//修订号
         typename value_type::id_type    _next_id = 0;
         index_type                      _indices;                  //indice：索引集？
         uint32_t                        _size_of_value_type = 0;
         uint32_t                        _size_of_this = 0;
   };
```

#### 定义chainbase命名空间中的结构体
* strcmp_less:判断字符串长度是否更短
* object：对象
* get_index_type：通过对象类型查找索引类型

```C++
   //字符穿长度比较
   struct strcmp_less{};
   
   //对象
   template<uint16_t TypeNumber, typename Derived>
   struct object
   {
      typedef oid<Derived> id_type;					//对象ID类型
      static const uint16_t type_id = TypeNumber;	//类型id
   };
  
   //该类应使用宏SET_INDEX_TYPE完善，从而可以查询索引类型
   template<typename T>
   struct get_index_type {};
```
