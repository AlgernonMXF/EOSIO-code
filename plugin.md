# plugin - 插件
定义了所有插件的基类：abstract_plugin
同时声明了plugin类和application类

> inside namespace appbase

* **class**
  ```C++
  
  class application;
  
  class abstract_plugin
  {
  public:
         enum state {
            registered,		// the plugin is constructed but doesn't do anything
            initialized,	// the plugin has initialized any state required but is idle
            started,		//the plugin is actively running
            stopped		// the plugin is no longer running
         };

         virtual ~abstract_plugin(){}
         virtual state get_state()const = 0;
         virtual const std::string& name()const  = 0;
         ///配置，在子类中实现
         virtual void set_program_options( options_description& cli, options_description& cfg ) = 0;         
         virtual void initialize(const variables_map& options) = 0;	///初始化，在plugin中重写        
         virtual void startup() = 0;					///启动，在plugin中重写     
         virtual void shutdown() = 0;					///关闭，在plugin中重写
  }
  
  class plugin;
  ```
  
 * **function**
 	```C++
	application& app();
  	```
  
  
