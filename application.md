# application - 插件管理器

定义了插件管理器类application，以及插件类plugin

> inside namespace appbase
* **namespace**
	```C++
	namespace bpo = boost::program_options
	namespace bfs = boost::filesystem
	```
	
* **class**
	```C++
	class application
	{
	public:
		~application();								//析构函数
		
		void set_version(uint64_t version);					//设置版本号（谁的？）
		uint64_t version() const;						//返回版本号
		void set_default_datat_dir(const bfs::path& data_dir = 'data-dir');	//设置默认数据目录
		bfs::path data_dir() const;						//返回数据目录
		void set_default_config_dir(const bfs::path& config_dir = 'etc');	//设置默认config目录
		bfs::path config_dir() const;						//返回config目录
		bfs::path get_logging_conf() const;					//返回登录设置位置
		
		template<typename... Plugin>
		//初始化插件，内部调用find_plugin通过插件名找到插件，然后将找到的插件数组传递给initialize——impl进行初始化
		//参数为插件列表
		//若插件未初始化，返回true；否则返回false
		bool initialize(int argc, char** argv) {return initialize_impl(argc, argv, {find_plugin<Plugin>()...});}
		
		void startup();
		void shutdown();
		void exec();
		void quit();
		static application& instance();
		abstract_plugin* find_plugin(const string& name) const;			//寻找插件，返回抽象插件指针
		abstract_plugin& get_plugin(const string& name) const;			//寻找插件，返回抽象插件地址
		
		template<typename Plugin>
		//注册插件，先查找插件是否存在；若不存在则创建新插件并添加到map对象plugins中，同时注册插件自己的依赖
		auto& register_plugin();	
		Plugin* find_plugin() const;		//查找并返回插件数组（全部？）
		Plugin& get_plugin() const;		//查找并返回插件数组，内部调用find_plugin，返回plugin引用
		//Fetch a reference to the method declared by the passed in type
		auto get_method() -> std::enable_if_t<is_method_decl<MethodDecl>::value, typename MethodDecl::method_type&>;
		//Fetch a reference to the channel declared by the passed in type
		auto get_channel() -> std::enable_if_t<is_channel_decl<ChannelDecl>::value, typename ChannelDecl::channel_type&>
		
		boost::asio::io_service& get_io_service() {return *io_server}
		
	protected:
		friend class plugin;			//友元类
		//插件初始化的实现
		bool initialize_impl(int argc, char** argv, vector<abstract_plugin*> autostart_plugins);
		
		//插件初始化和启动；将插件加入相应列表
		void plugin_initialized(abstract_plugin& plug) {initialized_plugins.push_back(&plug);}
		void plugin_started(abstract_plugin& plug){ running_plugins.push_back(&plug); }
	
	private:
		application();				//构造函数，私有原因：application为单例类
		
		//成员参数
		map<string, std::unique_ptr<abstract_plugin>>	plugins;		//记录所有注册的插件
		vector<abstract_plugin*>			initialized_plugins;	//已初始化的插件列表
		vector<abstract_plugin*>		        running_plugins;	//正在运行的插件列表
			
		map<std::type_index, erased_method_ptr>		methods;		//
		map<std::type_index, erased_channel_ptr>	channels;
		
		std::shared_ptr<boost::asio::io_service>  io_serv;

        	void set_program_options();
        	void write_default_config(const bfs::path& cfg_file);
        	void print_default_config(std::ostream& os);
        	std::unique_ptr<class application_impl> my;	
	}
	
	//插件类，继承abstract_plugin
	class plugin : public abstract_plugin
	{
	public:
		plugin():_name(boost::core::demangle(typeid(Impl).name())){}		//构造函数
         	virtual ~plugin(){}							//析构函数

		virtual state get_state()const override         { return _state; }	//获得插件状态state
         	virtual const std::string& name()const override { return _name; }	//获得插件名

         	virtual void register_dependencies();					//注册依赖插件
		virtual void initialize(const variables_map& options) override;		//初始化插件
		virtual void startup() override;					//启动插件
		virtual void shutdown() override;					//关闭插件
		
	protected:
		plugin(const string& name) : _name(name){}				//构造函数
		
	private:
		state _state = abstract_plugin::registered;				//插件状态：已注册?
		std::string _name;							//插件名
         }
	
	protected:
	
	}
	```
	
* **function**
	```C++
	application& app()
	```
