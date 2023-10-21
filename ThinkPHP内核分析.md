> 这是对tp5.0.24的分析，只简单的分析了框架"如何跑起来的原理"，并且只分析了我认为重要的内容，忽略掉了很多东西，也有很多的错误。
>
> 但是只要动态跟进去看个半天，就能知道tp5的运行流程了。再好的文字分析都不如进去看一遍。



# 流程

## 注册自动加载

> \\core\\library\\think\\loader.php
>
> \think\Loader::register();

### 懒惰的类库预定加载

> 默认注册think\\Loader::autoload

```php
spl_autoload_register($autoload ?: 'think\\Loader::autoload', true, true);
#这里有一点很巧妙，就是注册的是think\\Loader::autoload，按理说应该找不到目标方法的。但是加载的却是library\\think\\Loader::autoload。这是因为此时只是注册了自动加载的方法，等到后面做了命名空间映射，将think映射为\\library\\think后，就自然找到了library\\think\\Loader::autoload方法。
```

### Composer自动加载

```php
is_dir('/www/vender/composer')
#可以看到如果此文件夹存在的话，便取出最后一个类名，也就是刚包含的那个类
require VENDOR_PATH . 'composer' . DS . 'autoload_static.php';
$declaredClass = get_declared_classes();
$composerClass = array_pop($declaredClass);
#判断取出的类中是否包含这几个属性,结果为[prefixLengthsPsr4,prefixDirsPsr4]
foreach (['prefixLengthsPsr4', 'prefixDirsPsr4', 'fallbackDirsPsr4', 'prefixesPsr0', 'fallbackDirsPsr0', 'classMap', 'files'] as $attr) {
	if (property_exists($composerClass, $attr)) {
    	self::${$attr} = $composerClass::${$attr};
    }
}
```

### 命名空间映射

```php
self::addNamespace([
	'think'    => LIB_PATH . 'think' . DS,
    'behavior' => LIB_PATH . 'behavior' . DS,
    'traits'   => LIB_PATH . 'traits' . DS,
]);
#在addNamespace内后无论数组还是普通变量，都会进入addPsr4("$namespace\\","过滤path的双斜杠为单斜杠",true)
    public static function addNamespace($namespace, $path = '')
    {
        if (is_array($namespace)) {
            foreach ($namespace as $prefix => $paths) {
                self::addPsr4($prefix . '\\', rtrim($paths, DS), true);
            }
        } else {
            self::addPsr4($namespace . '\\', rtrim($path, DS), true);
        }
    }
#addPsr4
如果 $prefix 参数为空，表示要注册根命名空间的目录路径。这时，会将传入的 $paths 参数与已注册的根命名空间目录路径进行合并，然后根据 $prepend 参数决定是将新的目录路径放在前面还是后面。最终，将合并后的目录路径数组赋值给 self::$fallbackDirsPsr4 属性，作为根命名空间的目录路径映射。

如果 $prefix 参数不为空且该命名空间尚未注册过，表示要注册新的命名空间及其目录路径。此时，会计算出命名空间 $prefix 的长度，并将其作为键存储在 self::$prefixLengthsPsr4 数组中，值为 $prefix 的长度。然后，将传入的 $paths 参数转为数组，并将该数组赋值给 self::$prefixDirsPsr4[$prefix]，以建立命名空间与目录路径的映射关系。

如果 $prefix 参数不为空且该命名空间已经注册过，表示要向已有命名空间添加新的目录路径。这时，会将传入的 $paths 参数与已有的目录路径进行合并，然后根据 $prepend 参数决定是将新的目录路径放在前面还是后面。最终，将合并后的目录路径数组赋值给 self::$prefixDirsPsr4[$prefix] 属性，更新命名空间的目录路径映射。
    
    private static function addPsr4($prefix, $paths, $prepend = false)
    {
        if (!$prefix) {
            // Register directories for the root namespace.
            self::$fallbackDirsPsr4 = $prepend ?
            array_merge((array) $paths, self::$fallbackDirsPsr4) :
            array_merge(self::$fallbackDirsPsr4, (array) $paths);

        } elseif (!isset(self::$prefixDirsPsr4[$prefix])) {
            // Register directories for a new namespace.
            $length = strlen($prefix);
            if ('\\' !== $prefix[$length - 1]) {
                throw new \InvalidArgumentException(
                    "A non-empty PSR-4 prefix must end with a namespace separator."
                );
            }

            self::$prefixLengthsPsr4[$prefix[0]][$prefix] = $length;
            self::$prefixDirsPsr4[$prefix]                = (array) $paths;

        } else {
            self::$prefixDirsPsr4[$prefix] = $prepend ?
            // Prepend directories for an already registered namespace.
            array_merge((array) $paths, self::$prefixDirsPsr4[$prefix]) :
            // Append directories for an already registered namespace.
            array_merge(self::$prefixDirsPsr4[$prefix], (array) $paths);
        }
    }
```

### 类库映射

> loadComposerAutoloadFiles()



## 加载后进入程序正常流程

> 虽然注册自动加载后还要做各种操作，但是我的理解为注册完自动加载后已经进入了正常的流程了，只不过还有一大部分内核的操作没做完，这里举一个例子
>
> #注册错误和异常处理机制
> \think\Error::register();

```
注册完自动加载后立马调用了
	#注册错误和异常处理机制
	\think\Error::register();
方法，按理说这个命名空间应该是不存在的，但是前面做了空间映射，所以解析正常，但是在调用词方法前会进入autoload方法(因为前面注册了懒惰加载)进行操作:如果文件存在并且findfile()方法(PSR4检查)无误后进入__include_file包含类库映射后的文件。完成调用。
```

## 回到start.php执行应用

> App::run()->send();
>
> 在这里是加载app的公共配置文件，而在module方法里才是加载模块的配置

```php
1.init#/application/module/init.php	或者	/data/runtime/module/init.php
2.config#/application/config.php 或者 /application/module/config.php
3.数据库配置#/application/database.php 或者 /application/module/database.php
3.扩展配置#/application/extra下的所有文件，使用scan_dir得到数组
4.公共配置#\www\test/application/common.php    
5.语言包#/application/lang/cn.php
6.获得并且配置当前app的app_debug到环境变量
7.注册插件的跟命名空间为	weapp\
8.加载封装的函数#自己写的助手函数
#框架的助手函数
#自己定义的函数
9.路由解析操作    
    
#$config = self::initCommon();加载配置(如果self::$init为空，那就加载默认配置)
	#进入initCommon()后先做命名空间映射映射默认app//为application//，有点不太明白，前面不是做过命名空间映射了嘛
	#initCommon()内进入 $config = self::init();
		#如果存在/application/module/init.php文件则包含其(这里是不存在)，同理处理/data/runtime/module/init.php(也不存在)
        //上面不存在后就加载模块配置    （如果传入的module有值，就会破解加载module后加载module下的config.php）
        $config = Config::load(CONF_PATH . $module . 'config' . CONF_EXT);		
			if ('php' == $type) {
        		return self::set(include $config(/application/config.php或者/application/module/config.php), $name, $range);
			}
			        // 数组则表示批量设置
       		 if (is_array($name)) {
            	if (!empty($value)) {
                	self::$config[$range][$value] = isset(self::$config[$range][$value]) ?
                    	array_merge(self::$config[$range][$value], $name) :
                    	$name;
                	return self::$config[$range][$value];
            }
		#加载数据库配置
			$filename = 'application/' . $module . '/database.php';
            Config::load($filename, 'database');
				#同理
				if ('php' == $type) {
                	return self::set(include $file, $name, $range);
            	}
             #set方法内
             // 数组则表示批量设置
	        if (is_array($name)) {
    	        if (!empty($value)) {
        	        self::$config[$range][$value] = isset(self::$config[$range][$value]) ?
            	        array_merge(self::$config[$range][$value], $name) :
                	    $name;
                	return self::$config[$range][$value];
            	}
        #读取扩展目录下/application/extra的扩展配置文件，使用scan_dir得到数组后同上面的原理
        
		#加载公共配置文件\www\test/application/common.php或者\www\test/application/module/common.php                
		#加载当前模块语言包	/application/lang/cn.php
            if ($module) {
                Lang::load($path . 'lang' . DS . Request::instance()->langset() . EXT);
            }                
		return config::get()
    #根据返回的config做配置：
            #app_debug，这里是不是可以传入get参数设置app_debug????????????????很迷，等会儿试一下
             if (isset($_GET['app_debug'])) {
                Config::set('app_debug', $_GET['app_debug']);
            }
			#然后根据app_debug将配置加载到环境变量内
                self::$debug = Env::get('app_debug', Config::get('app_debug'));
                $show_error_msg = Config::get('show_error_msg'); // by 小虎哥
                if (!$show_error_msg) {
                    ini_set('display_errors', 'Off');
                } elseif (!IS_CLI) {
                    // 重新申请一块比较大的 buffer
                    if (ob_get_level() > 0) {
                        $output = ob_get_clean();
                    }

                    ob_start();

                    if (!empty($output)) {
                        echo $output;
                    }
                }
	#注册插件的命名空间，不知道是干什么的
		$config['root_namespace'] == weapp\;
		Config::set('root_namespace', $config['root_namespace']);
	#加载额外文件，主要是这三个文件
			0 = "/application/helper.php"			#自己写的助手函数
			1 = "/core/helper.php"					#框架的助手函数
			2 = "/application/function.php"			#自己定义的函数
            if (!empty($config['extra_file_list'])) {
                foreach ($config['extra_file_list'] as $file) {
                    $file = strpos($file, '.') ? $file : APP_PATH . $file . EXT;
                    if ( is_file($file) && !isset(self::$file[$file])) {
                        include $file;
                        self::$file[$file] = true;
                    }
                }
            }                
	#监听 app_init
    	Hook::listen('app_init');
                
	#routeCheck并且导入路由配置。核心是加载路由规则的那几段和后面几段
         #解析路由：默认使用老方法 ?m=&c=&a=
     			  #没有的话就使用parseUrl解析：/module/controller/action，成功解析。
	
                   public static function routeCheck($request, array $config)
                {
                    $path   = $request->path();
                    $depr   = $config['pathinfo_depr'];
                    $result = false;

                    // 路由检测
                    $check = !is_null(self::$routeCheck) ? self::$routeCheck : $config['url_route_on'];
                    if ($check) {
                        // 开启路由
                        if (is_file(RUNTIME_PATH . 'route.php')) {
                            // 读取路由缓存
                            $rules = include RUNTIME_PATH . 'route.php';
                            is_array($rules) && Route::rules($rules);
                        } else {
                            $files = $config['route_config_file'];
                            foreach ($files as $file) {
                                if (is_file(CONF_PATH . $file . CONF_EXT)) {
                                    // 导入路由配置
                                    $rules = include CONF_PATH . $file . CONF_EXT;
                                    is_array($rules) && Route::import($rules);
                                }
                            }
                        }

                        // 路由检测（根据路由定义返回不同的URL调度）
                        $result = Route::check($request, $path, $depr, $config['url_domain_deploy']);
                        $must   = !is_null(self::$routeMust) ? self::$routeMust : $config['url_route_must'];

                        if ($must && false === $result) {
                            // 路由无效
                            throw new RouteNotFoundException();
                        }
                    }

                    // 路由无效 解析模块/控制器/操作/参数... 支持控制器自动搜索
                    if (false === $result) {
                        //兼容以前的老方法 by 小虎哥
                        if(($m = $request->get('m')) && ($c = $request->get('c')) && ($a = $request->get('a')))
                        {
                            $result = ['type' => 'module', 'module' => [$m, $c, $a]];//兼容以前的3.2的老版本
                        }
                        else
                        {    // 路由无效 解析模块/控制器/操作/参数... 支持控制器自动搜索
                            $result = Route::parseUrl($path, $depr, $config['controller_auto_search']);
                        }
                    }
                    return $result;
                }                
	#将请求分发到对应控制器（终于进来了框架二次开发者可控制的操作）
		$request->dispatch($dispatch);
```

## 控制权转移

### module解析模块

> 加载module的配置

```php
public static function module($result, $config, $convert = null)
    #在里面初始化模块
    	$config       = self::init();
    	private static function init($module = '')
    		#如果模块下有init.php便包含它
    			include '/applicatino/module/init.php'
    		#否则加载模块默认配置
    			$config = Config::load('/application/module/config.php');
			#读取数据库配置文件
				$filename = '/application/module/database.php';
            	Config::load($filename, 'database');
			#读取扩展目录下的所有文件
				scandir('/application/module/extra');
			#加载行为扩展文件(用来注册HOOK的)
				Hook::import(include '/application/module.tags.php');
				#从数据库加载行为扩展文件
                	$weappRow = \think\Db::name('weapp')->field('code')->where(['status'=> 1,])->cache(true, null, "weapp")->order('sort_order asc, id asc')->select(); 
				#加载behavior行为扩展文件
					Hook::import(include '/core/library/think/behavior/module/tags.php');
			#加载应用程序下的公共配置文件/或者模块下的文件
				include '/application/common.php';	|| include '/application/module/common.php';
				#在公共配置文件内会加载\extend\function.php
					include '\extend\function.php';
			#加载当前模块下的语言包，如果存在的话
			
	#接受get传入的app_debug，并且获取app_debug
		Config::set('app_debug', $_GET['app_debug']);
	#设置当前APP实例的app_debug
		self::$debug = Env::get('app_debug', Config::get('app_debug'));
	#注册插件的跟命名空间
		Config::set('root_namespace', $config['root_namespace']);		//这里为{'weapp'=>'weapp\'}
	#如果上一步设置完毕后就将其进行命名空间映射
		if (!empty($config['root_namespace'])) {
        	Loader::addNamespace($config['root_namespace']);
		}		
	#加载额外文件(一些助手函数之类的)
		'E:\Desktop\CodeAudit\www\test/application/helper.php';		//二开者写的助手函数
		'E:\Desktop\CodeAudit\www\test\core\helper.php'				//框架助手函数
        'E:\Desktop\CodeAudit\www\test/application/function.php'	//二开者写的函数
	#监听app_init
		Hook::listen('app_init');
#出来后到run方法监听app_dispatch
	Hook::listen('app_dispatch', self::$dispatch);
#后面有一大堆操作，但是我觉得不重要，而且很多都是重复调用过的方法，整个逻辑我看不明白，但是不影响分析。上面的分析也有很大一部分是错误的(在不结合动态调试来看根本看不明白，跟没看一样)。动态跟一遍脑子里有个印象就好了。

#在exec方法一大堆操作，核心在这
    protected static function exec($dispatch, $config)
    {
        switch ($dispatch['type']) {
            case 'redirect': // 重定向跳转
                $data = Response::create($dispatch['url'], 'redirect')
                    ->code($dispatch['status']);
                break;
            case 'module': // 模块/控制器/操作	
                $data = self::module(					//在这个方法里解析多模块，模块绑定，初始化模块self::init($module);也就是上面分析的一堆加载配置的，模块缓存请求检查，设置默认过滤机制！！获取控制器名和操作名，监听Hook::listen('module_init', $request);执行操作方法$request->action($actionName);判断是否是invokeMethod(),然后return $reflect->invokeArgs(isset($class) ? $class : null, $args);就直接进入了操作。。。
                    $dispatch['module'],
                    $config,
                    isset($dispatch['convert']) ? $dispatch['convert'] : null
                );
                break;
            case 'controller': // 执行控制器操作
                $vars = array_merge(Request::instance()->param(), $dispatch['var']);
                $data = Loader::action(
                    $dispatch['controller'],
                    $vars,
                    $config['url_controller_layer'],
                    $config['controller_suffix']
                );
                break;
            case 'method': // 回调方法
                $vars = array_merge(Request::instance()->param(), $dispatch['var']);
                $data = self::invokeMethod($dispatch['method'], $vars);
                break;
            case 'function': // 闭包
                $data = self::invokeFunction($dispatch['function']);
                break;
            case 'response': // Response 实例
                $data = $dispatch['response'];
                break;
            default:
                throw new \InvalidArgumentException('dispatch type not support');
        }

        return $data;
    }
#然后就是进入Response.php进行响应了。
```



# 容器注入IOC 门面模式Facade

1. 创建低级类，类内包含一些方法

2. 创建容器类，类内包含bind方法实现将低级类的创建闭包函数并存入$instance数组(这里面现在不是一个对象，而使一个能创建对象的函数)，

3. 创建门面类，类内包含初始化方法和一些间接调用低级类的方法。

    类内包含$Container属性

    使用Facade::initialize($容器对象)将容器对象赋值给$Container属性。然后门面类里写各种方法，来调用自身属性里的static::$Container->make('低级类')->display()

    make方法来调用闭包函数进行实例化对象，然后执行对应方法。

```php
<?php 

//数据库操作类
class Db
{
	//数据库连接
	public function connect()
	{
		return '数据库连接成功<br>';
	}
}

//数据验证类
class Validate
{
	//数据验证
	public function check()
	{
		return '数据验证成功<br>';
	}
}

//视图
class View
{
	//内容输出
	public function display()
	{
		return '用户登录成功';
	}
}

//一、创建容器类
class Container
{
	//创建属性,用空数组初始化,该属性用来保存类与类的实例化方法
	public $instance = [];

	//初始化实例数组,将需要实例化的类,与实例化的方法进行绑定
	public function bind($abstract, Closure $process)
	{
		//键名为类名,值为实例化的方法
		$this->instance[$abstract] = $process;
		
		//$this->instance['db'] = new Db();
	}

	//创建类实例
	public function make($abstract, $params=[])
	{
		return call_user_func_array($this->instance[$abstract],[]);
	}
}



//三、容器依赖：将容器对象,以参数的方式注入到当前工作类中
class Facade
{
	//创建成员属性保存容器对象
	protected static $container = null;

	//创建初始化方法为容器对象赋值
	public static function initialize(Container $container)
	{
		static::$container = $container;
		
		//static::$container = new container();
	}
        
        //连接数据库
        public static function connect()
	{
		return static::$container->make('db')->connect();
	}

	//用户数据验证
	public static function check()
	{
		return static::$container->make('validate')->check();
	}

	//输出提示信息
	public static function display()
	{
		return static::$container->make('view')->display();
	}
}


//二、服务绑定: 将类实例注册到容器中
$container = new Container(); 

//将Db类绑定到容器中
$container->bind('db', function(){
	return new Db();
});

//将Validate类实例绑定到容器中
$container->bind('validate', function(){
	return new Validate();
});

//将View类实例绑定到容器中
$container->bind('view', function(){
	return new View();
});


//客户端调用

//初始化类门面类中的容器对象
Facade::initialize($container);

//静态统一调用内部的方法(无须重复注入依赖容器对象啦,实现了细节隐藏,通用性更强)
echo Facade::connect();
echo Facade::check();
echo Facade::display();
```

# APP架构

> 因为是MVC架构，所以控制器通常不处理逻辑，而是充当一个 模型与视图 之间的粘合剂。
>
> thinkphp也不建议在控制器中做逻辑处理，只是做一个间接调用的角色。
>
> 这点很重要，要不然代码审计都不知道审计什么东西。(当然是对模型进行审计)

### 控制器 调用 模型

> 在这里使用model助手函数将modelObj赋值为模型目录下的Minipro0001模型了。(model助手函数自动识别的)
>
> 然后使用$this->modelObj->getGlobalsConf()调用模型

```php
    public function __construct(){
        parent::__construct();
        $this->nid = CONTROLLER_NAME;
        $this->modelObj = model('Minipro0001');
    }

    /**
     * 全局常量API
     */
    public function globals()
    {
        $data = $this->modelObj->getGlobalsConf();

        exit(json_encode($data));
    }
```



# HOOK

> Hook.php中的Hook方法包含
>
> - $tags\[标签名]\[行为]
>
> - add方法接受一个标签名和行为，接着往$tag中添加一个新成员。
> - import是接受一个数组的标签-行为，然后循环调用add方法导入。
> - get则是获取指定标签的行为。
> - listen是监听标签的行为，当标签有行为的时候则会调用exec执行。
> - exec则是执行标签的行为。

### 实例

> 在APP::run中

```php
Hook::listen('app_begin', $dispatch);

#这个时候app_begin标签就已经被触发，app_begin标签所对应的行为就会被执行。
#不过在执行之前Hook类中必须要有app_begin标签和其行为，他们是在APP::init中被导入的：
if (is_file(CONF_PATH . $module . 'tags' . EXT)) {
                Hook::import(include CONF_PATH . $module . 'tags' . EXT);
 }
```

# 路由

## 路由解析

> 在没开启url_route_on的情况下，默认使用普通路由模式
>
> /module/controller/action/param

### request实例

在run方法中，将获取一个request实例

```php
is_null($request) && $request = Request::instance();
```

### routeCheck检查路由

> 在这里通过Request::path()来调用path_info()原生函数来解析并且返回$path数组，再判断url_route_on是否开启，然后调用Route::parseUrl方法来解析出	模块/控制器/方法/参数，实际上就是简单的弹出了 module/controller/action/param

```php
#routeCheck

public static function routeCheck($request, array $config)
    {
        $path   = $request->path();
        $depr   = $config['pathinfo_depr'];
        $result = false;
        // 路由检测
        $check = !is_null(self::$routeCheck) ? self::$routeCheck : $config['url_route_on'];
        if ($check) {
            // 开启路由
            //。。。
        }
        if (false === $result) {
            // 路由无效 解析模块/控制器/操作/参数... 支持控制器自动搜索
            $result = Route::parseUrl($path, $depr, $config['controller_auto_search']);
        }
        return $result;
    }
#parseUrl
public static function parseUrl($url, $depr = '/', $autoSearch = false)
    {
		...
        if (isset($path)) {
            // 解析模块
            $module = Config::get('app_multi_module') ? array_shift($path) : null;
            if ($autoSearch) {
                ...
                } else {
                    $controller = array_shift($path);
                }
            } else {
                // 解析控制器
                $controller = !empty($path) ? array_shift($path) : null;
            }
            // 解析操作
            $action = !empty($path) ? array_shift($path) : null;
            // 解析额外参数
            self::parseUrlParams(empty($path) ? '' : implode('|', $path));
            // 封装路由
            $route = [$module, $controller, $action];
            ...
        }
        return ['type' => 'module', 'module' => $route];
    }
```

### parseUrl解析路由

> 在这里检查访问对应的控制器是否存在，如果不存在就返回默认控制器。

```php
    public static function parseUrl($url, $depr = '/', $autoSearch = false)
    {

        if (isset(self::$bind['module'])) {
            $bind = str_replace('/', $depr, self::$bind['module']);
            // 如果有模块/控制器绑定
            $url = $bind . ('.' != substr($bind, -1) ? $depr : '') . ltrim($url, $depr);
        }
        $url              = str_replace($depr, '|', $url);
        list($path, $var) = self::parseUrlPath($url);
        $route            = [null, null, null];
        if (isset($path)) {
            // 解析模块
            $module = Config::get('app_multi_module') ? array_shift($path) : null;
            if ($autoSearch) {
                // 自动搜索控制器
                $dir    = APP_PATH . ($module ? $module . DS : '') . Config::get('url_controller_layer');
                $suffix = App::$suffix || Config::get('controller_suffix') ? ucfirst(Config::get('url_controller_layer')) : '';
                $item   = [];
                $find   = false;
                foreach ($path as $val) {
                    $item[] = $val;
                    $file   = $dir . DS . str_replace('.', DS, $val) . $suffix . EXT;
                    $file   = pathinfo($file, PATHINFO_DIRNAME) . DS . Loader::parseName(pathinfo($file, PATHINFO_FILENAME), 1) . EXT;
                    if (is_file($file)) {
                        $find = true;
                        break;
                    } else {
                        $dir .= DS . Loader::parseName($val);
                    }
                }
                if ($find) {
                    $controller = implode('.', $item);
                    $path       = array_slice($path, count($item));
                } else {
                    $controller = array_shift($path);
                }
            } else {
                // 解析控制器
                $controller = !empty($path) ? array_shift($path) : null;
            }
            // 解析操作
            $action = !empty($path) ? array_shift($path) : null;
            // 解析额外参数
            self::parseUrlParams(empty($path) ? '' : implode('|', $path));
            // 封装路由
            $route = [$module, $controller, $action];
            // 检查地址是否被定义过路由
            $name  = strtolower($module . '/' . Loader::parseName($controller, 1) . '/' . $action);
            $name2 = '';
            if (empty($module) || isset($bind) && $module == $bind) {
                $name2 = strtolower(Loader::parseName($controller, 1) . '/' . $action);
            }

            if (isset(self::$rules['name'][$name]) || isset(self::$rules['name'][$name2])) {
                throw new HttpException(404, 'invalid request:' . str_replace('|', $depr, $url));
            }
        }
        return ['type' => 'module', 'module' => $route];
    }
    
```



## 路由注册

> 其它方法都是通过封装Route::rule得来的。注册路由最常用的是使用Route::rule，不过TP内建也封装了其他的路由注册方法。这些方法基本上都是对Route::rule加上一层封装的，比如any，get，post等等之类的。

## 路由绑定到控制器

> 在解析出 module/controller/action/param 后，将进行checkUrlBind，并且在这个方法内调用【bindToClass；bindToController；bindToNamespace】将得出的路由结果绑定到	/类/控制器/命名空间，随后返回，并经过	switch ($dispatch['type']) {}做一些列操作

> 该方法会先检测self::$bind，然后取出type，根据类型判断是绑定到类，控制器或者是命名空间，然后调用响应的方法。Route::bind方法会设置self::$bind成员变量，因此设置之后这里便能够读取了。

```php
1.#
private static function checkUrlBind(&$url, &$rules, $depr = '/')
    {
        if (!empty(self::$bind)) {
            $type = self::$bind['type'];
            $bind = self::$bind[$type];
            // 记录绑定信息
            App::$debug && Log::record('[ BIND ] ' . var_export($bind, true), 'info');
            // 如果有URL绑定 则进行绑定检测
            switch ($type) {
                case 'class':
                    // 绑定到类
                    return self::bindToClass($url, $bind, $depr);
                case 'controller':
                    // 绑定到控制器类
                    return self::bindToController($url, $bind, $depr);
                case 'namespace':
                    // 绑定到命名空间
                    return self::bindToNamespace($url, $bind, $depr);
            }
        }
        return false;
    }

2.#
public static function bindToController($url, $controller, $depr = '/')
    {
        $url    = str_replace($depr, '|', $url);
        $array  = explode('|', $url, 2);
        $action = !empty($array[0]) ? $array[0] : Config::get('default_action');
        if (!empty($array[1])) {
            self::parseUrlParams($array[1]);
        }
        return ['type' => 'controller', 'controller' => $controller . '/' . $action];
    }
```

# Request

### 特性

- 单例模式，构造函数不能通过外部访问
- 对象通过 Request::instance方法访问
- 基本上对$_SERVER的封装

### 属性

```php
#object 对象实例 
protected static $instance;


#  array 当前调度信息
protected $dispatch = [];
protected $module;
protected $controller;
protected $action;

#请求参数
protected $param   = [];
protected $get = [];
protected $post= [];
protected $request = [];
protected $route   = [];
protected $put;
protected $session = [];
protected $file= [];
protected $cookie  = [];
protected $server  = [];
protected $header  = [];

#array 当前路由信息
protected $routeInfo = [];

#Hook扩展方法
protected static $hook = [];

#过滤规则
protected $filter;

#还有一些其它的东西：url，pathinfo，mothod等待
```

## 方法

### 构造函数

> 初始化过滤方法，保存php://input

```php
protected function __construct($options = [])
    {
        foreach ($options as $name => $item) {
            if (property_exists($this, $name)) {
                $this->$name = $item;
            }
        }
        if (is_null($this->filter)) {
            $this->filter = Config::get('default_filter');
        }

        // 保存 php://input
        $this->input = file_get_contents('php://input');
    }
```

### instance

> 通过instance获取Request实例，因为Request是单例模式，所以这是主要的够获得Request对象的方法

```php
public static function instance($options = [])
```

### create

> 解析出以下这下变量，并且依据这些通过构造方法实例化一个新的Request对象
>
> 在特定场景下创建一个新的实例，可以使用 `create` 方法创建新的实例，否则可以使用 `instance` 方法获取之前创建的实例。

```php
    $server['REQUEST_URI']  
    $server['QUERY_STRING'] 
    $options['cookie']      
    $options['param']       
    $options['file']        
    $options['server']      
    $options['url']         
    $options['baseUrl']     
    $options['pathinfo']    
    $options['method']      
    $options['domain']      
    $options['content']     
self::$instance         = new self($options);
return self::$instance;
```

#### input方法的封装系列

```php
#一些方法都会调用到input方法，也就是说都是对input方法的封装
    [request,post,get,put,delete,session,patch,server,file,env,]
```

### header

> 解析content-type 和 content-length

### input

> input对传入数据做了安全处理，其本质是调用了filterValue方法。并且调用了typeCast方法做了强制类型转换

```php
array_walk_recursive($data, [$this, 'filterValue'], $filter);			//如果是数组的话
$this->filterValue($data, $name, $filter);								//如果非数组的话
$this->typeCast($data, $type);
```

### 获取和设置过滤器

> Request::getFilter只是获取和设置GET请求的参数。然后Request::filter可以获取设置所有类型的请求参数

`filter`

> 设置或获取  过滤规则

```php
public function filter($filter = null)
    {
        if (is_null($filter)) {
            return $this->filter;
        } else {
            $this->filter = $filter;
        }
    }
```

`getFilter`

> 方法用于获取最终要应用的过滤器数组，并在其中添加了默认值(无论配置中是否有过滤器，都会加载传入的默认的过滤器)。

```php
传入的过滤器为空:#那么就使用传入的default过滤器
传入的过滤器非空:#如果传入的为空就返回当前规则，否则返回处理后的传入的规则。
#这里使用到了空值合并运算符 条件?:false的话为此表达式
#如果条件为1那就保持不变，如果条件为0，采用表达式1
#$a = 1 ?:0			#在这里$a一定会是1
#$a = 0 ?:1			#在这里一定也会是1
这里是一定保持纯如的过滤器。


如果 $filter 为字符串且不包含斜杠 /，则将其按逗号 , 拆分为数组。
否则，将 $filter 强制转换为数组(强制转换默认以/分割)。
    
最后，将 $default 参数添加到过滤器数组的末尾，并返回最终的过滤器数组。
    
    
    protected function getFilter($filter, $default)
    {
        if (is_null($filter)) {
            $filter = [];
        } else {
            $filter = $filter ?: $this->filter;
            if (is_string($filter) && false === strpos($filter, '/')) {
                $filter = explode(',', $filter);
            } else {
                $filter = (array) $filter;
            }
        }

        $filter[] = $default;
        return $filter;
    }    
```



### filterValue 过滤数据

> 传入的$filters是一个二维数组，在弹出最后的一个元素后，将其进行foreach

```php
#三种大情况 + 绝对操作
1.是方法或函数:#就使用 call_user_func(过滤器,要过滤的内容)

2.不是方法的是一个普通的变量(标量值):#并且判断里面有没有/，如果有的话那么就是一个正则表达式 然后进行正则匹配。如果$filtersz正则表达式没匹配到传入的$value，那么$value等于默认的值

3.	是整形:#把它当作过滤器ID，使用原生的filter_var过滤$value。
	字符串:#按照字符串名字找到过滤器id(使用原生函数filter_id完成)，然后使用filter_var过滤器完成过滤

4.末尾:#最后做上一个表达式的过滤(在表达式尾部加上一个空格)


private function filterValue(&$value, $key, $filters)
    {
        $default = array_pop($filters);
        foreach ($filters as $filter) {
            if (is_callable($filter)) {
                // 调用函数或者方法过滤
                $value = call_user_func($filter, $value);
            } elseif (is_scalar($value)) {
                if (false !== strpos($filter, '/')) {
                    // 正则过滤
                    if (!preg_match($filter, $value)) {
                        // 匹配不成功返回默认值
                        $value = $default;
                        break;
                    }
                } elseif (!empty($filter)) {
                    // filter函数不存在时, 则使用filter_var进行过滤
                    // filter为非整形值时, 调用filter_id取得过滤id
                    $value = filter_var($value, is_int($filter) ? $filter : filter_id($filter));
                    if (false === $value) {
                        $value = $default;
                        break;
                    }
                }
            }
        }
        return $this->filterExp($value);
    }

```



### filterExp

> 过滤表单中的  表达式，将表达式后加上一个空格

```php
if (is_string($value) && preg_match('/^(EXP|NEQ|GT|EGT|LT|ELT|OR|XOR|LIKE|NOTLIKE|NOT LIKE|NOT BETWEEN|NOTBETWEEN|BETWEEN|NOT EXISTS|NOTEXISTS|EXISTS|NOT NULL|NOTNULL|NULL|BETWEEN TIME|NOT BETWEEN TIME|NOTBETWEEN TIME|NOTIN|NOT IN|IN)$/i', $value)) {
	$value .= ' ';
}
```

### typeCast

> 强制类型转换，只是根据传入的数据和传入的类型做简单的类型转换罢了，只是实现了switch。没什么别的了

```php
  private function typeCast(&$data, $type)
    {
        switch (strtolower($type)) {
            // 数组
            case 'a':
                $data = (array) $data;
                break;
            // 数字
            case 'd':
                $data = (int) $data;
                break;
            // 浮点
            case 'f':
                $data = (float) $data;
                break;
            // 布尔
            case 'b':
                $data = (boolean) $data;
                break;
            // 字符串
            case 's':
            default:
                if (is_scalar($data)) {
                    $data = (string) $data;
                } else {
                    throw new \InvalidArgumentException('variable type error：' . gettype($data));
                }
        }
    }
```

### ip

> 获取客户端的IP并且验证合法性

```php
$httpAgentIp = Config::get('http_agent_ip');

        if ($httpAgentIp && isset($_SERVER[$httpAgentIp])) {
            $ip = $_SERVER[$httpAgentIp];
        } elseif ($adv) {
            if (isset($_SERVER['HTTP_X_FORWARDED_FOR'])) {
                $arr = explode(',', $_SERVER['HTTP_X_FORWARDED_FOR']);
                $pos = array_search('unknown', $arr);
                if (false !== $pos) {
                    unset($arr[$pos]);
                }
                $ip = trim(current($arr));
            } elseif (isset($_SERVER['HTTP_CLIENT_IP'])) {
                $ip = $_SERVER['HTTP_CLIENT_IP'];
            } elseif (isset($_SERVER['REMOTE_ADDR'])) {
                $ip = $_SERVER['REMOTE_ADDR'];
            }
        } elseif (isset($_SERVER['REMOTE_ADDR'])) {
            $ip = $_SERVER['REMOTE_ADDR'];
        }
        // IP地址合法验证
        $long = sprintf("%u", ip2long($ip));
        $ip   = $long ? [$ip, $long] : ['0.0.0.0', 0];
        return $ip[$type];
```

### 

### dispatch调度器

> 设置或者获取当前请求的调度信息
>
> 调度器也就是   进行路由匹配后将请求 转到 对应的 模块/控制器/方法 ，并且传入参数。
>
> 这里使用的是	HTTP 路由调度器（HttpDispatcher）

```php
    public function dispatch($dispatch = null)
    {
        
        if (!is_null($dispatch)) {
            $this->dispatch = $dispatch;
        }
        return $this->dispatch;
    }
```

### module

> 设置或获取当前module

```php
    public function module($module = null)
    {
        if (!is_null($module)) {
            $this->module = $module;
            return $this;
        } else {
            return $this->module ?: '';
        }
    }
```

### controller

> 设置或获取当前controller

```php
    public function controller($controller = null)
    {
        if (!is_null($controller)) {
            $this->controller = $controller;
            return $this;
        } else {
            return $this->controller ?: '';
        }
    }
```

### action

> 设置或获取当前action

```php
    public function action($action = null)
    {
        if (!is_null($action) && !is_bool($action)) {
            $this->action = $action;
            return $this;
        } else {
            $name = $this->action ?: '';
            return true === $action ? $name : strtolower($name);
        }
    }
```

### token加密算法的设置

> 默认为md5

```php
    public function token($name = '__token__', $type = 'md5')
    {
        $type  = is_callable($type) ? $type : 'md5';
        $token = call_user_func($type, $_SERVER['REQUEST_TIME_FLOAT']);
        if ($this->isAjax()) {
            header($name . ': ' . $token);
        }
        Session::set($name, $token);
        return $token;
    }
```

### 缓存

```php
   /**
     * 设置当前地址的请求缓存
     * @access public
     * @param string $key    缓存标识，支持变量规则 ，例如 item/:name/:id
     * @param mixed  $expire 缓存有效期
     * @param array  $except 缓存排除
     * @param string $tag    缓存标签
     * @return void
     */
    public function cache($key, $expire = null, $except = [], $tag = null)
    {
        if (!is_array($except)) {
            $tag    = $except;
            $except = [];
        }

        if (false !== $key && $this->isGet() && !$this->isCheckCache) {
            // 标记请求缓存检查
            $this->isCheckCache = true;
            if (false === $expire) {
                // 关闭当前缓存
                return;
            }
            if ($key instanceof \Closure) {
                $key = call_user_func_array($key, [$this]);
            } elseif (true === $key) {
                foreach ($except as $rule) {
                    if (0 === stripos($this->url(), $rule)) {
                        return;
                    }
                }
                // 自动缓存功能
                $key = '__URL__';
            } elseif (strpos($key, '|')) {
                list($key, $fun) = explode('|', $key);
            }
            // 特殊规则替换
            if (false !== strpos($key, '__')) {
                $key = str_replace(['__MODULE__', '__CONTROLLER__', '__ACTION__', '__URL__', ''], [$this->module, $this->controller, $this->action, md5($this->url(true))], $key);
            }

            if (false !== strpos($key, ':')) {
                $param = $this->param();
                foreach ($param as $item => $val) {
                    if (is_string($val) && false !== strpos($key, ':' . $item)) {
                        $key = str_replace(':' . $item, $val, $key);
                    }
                }
            } elseif (strpos($key, ']')) {
                if ('[' . $this->ext() . ']' == $key) {
                    // 缓存某个后缀的请求
                    $key = md5($this->url());
                } else {
                    return;
                }
            }
            if (isset($fun)) {
                $key = $fun($key);
            }

            if (strtotime($this->server('HTTP_IF_MODIFIED_SINCE')) + $expire > $_SERVER['REQUEST_TIME']) {
                // 读取缓存
                $response = Response::create()->code(304);
                throw new \think\exception\HttpResponseException($response);
            } elseif (Cache::has($key)) {
                list($content, $header) = Cache::get($key);
                $response               = Response::create($content)->header($header);
                throw new \think\exception\HttpResponseException($response);
            } else {
                $this->cache = [$key, $expire, $tag];
            }
        }
    }

    /**
     * 读取请求缓存设置
     * @access public
     * @return array
     */
    public function getCache()
    {
        return $this->cache;
    }
```

### 设置当前请求绑定的对象实例

> 存储各个实例的标识符和实例的对应关系的
>
> 很简单：给已经实例化了的对象取个名字，然后放入bind方法。形成一个对应关系，方便在请求处理过程中访问和使用这些对象实例。

```php
$user = new User('John', 'Doe');
$request->bind('用户1', $user);
```

```php
    /**
     * 设置当前请求绑定的对象实例
     * @access public
     * @param string|array $name 绑定的对象标识
     * @param mixed        $obj  绑定的对象实例
     * @return mixed
     */
    public function bind($name, $obj = null)
    {
        if (is_array($name)) {
            $this->bind = array_merge($this->bind, $name);
        } else {
            $this->bind[$name] = $obj;
        }
    }

    public function __set($name, $value)
    {
        $this->bind[$name] = $value;
    }

    public function __get($name)
    {
        return isset($this->bind[$name]) ? $this->bind[$name] : null;
    }

    public function __isset($name)
    {
        return isset($this->bind[$name]);
    }
}

```















```
scheme()：获取请求的协议，比如http或者https
isSsl():判断是否使用了ssl
isAjax():判断是否是ajax请求，通过判断$_SERVER['HTTP_X_REQUESTED_WITH']
ip()：获取ip
```







# 模板

> 模板的本质就是将一段文本导入，然后用变量替换其特定的字符串。

## 模板内容获取

> 先是使用file_get_contents($template)来将模板导入进来

```php
private function parseTemplateName($templateName)
    {
        ...
            if ($template) {
                // 获取模板文件内容
                $parseStr .= file_get_contents($template);
            }
       ...
    }
```

## 模板定位

> 解析出当前控制器的方法使用哪个模板

```php
   private function parseTemplateFile($template)
    {
        if ('' == pathinfo($template, PATHINFO_EXTENSION)) {
            if (strpos($template, '@')) {
                // 跨模块调用模板
                $template = str_replace(['/', ':'], $this->config['view_depr'], $template);
                $template = APP_PATH . str_replace('@', '/' . basename($this->config['view_path']) . '/', $template);
            } else {
                $template = str_replace(['/', ':'], $this->config['view_depr'], $template);
                $template = $this->config['view_path'] . $template;
            }
            $template .= '.' . ltrim($this->config['view_suffix'], '.');
        }

        if (is_file($template)) {
            // 记录模板文件的更新时间
            $this->includeFile[$template] = filemtime($template);
            return $template;
        } else {
            throw new TemplateNotFoundException('template not exists:' . $template, $template);
        }
    }
```

## 模板缓存

> 将模板缓存在内存中

## 布局Layout

> 比如大部分页面都有一个相同的导航和一个相同的页脚，只有中间的内容不一样，这个时候我们就可以使用布局来实现，从而减少工作量。

### 布局定位

> 找到当前要使用的布局
>
> 可以看到如果没有开启layout_on配置的话  以及第一个if判断如果$content内含{\_\_NOLAYOUT\_\_}，就直接将其替换为空，也就是不进行布局
>
> 如果允许布局的话，便使用模板定位方法(传入布局名字)找到布局模板，然后将布局标识替换为读取出的布局模板内容。

```php
/*三种情况：
	layout_on为false										#不布局
	layout-on为on但是$content内包含{__NOLAYOUT__}			#不布局
    其它情况											 #进行布局解析
    */

if ($this->config['layout_on']) {
            if (false !== strpos($content, '{__NOLAYOUT__}')) {
                // 可以单独定义不使用布局
                $content = str_replace('{__NOLAYOUT__}', '', $content);
            } else {
                // 读取布局模板
                $layoutFile = $this->parseTemplateFile($this->config['layout_name']);
                if ($layoutFile) {
                    // 替换布局的主体内容
                    $content = str_replace($this->config['layout_item'], $content, file_get_contents($layoutFile));
                }
            }
        } else {
            $content = str_replace('{__NOLAYOUT__}', '', $content);
        }
```



## fetch

> 渲染方法里会先判断是否使用了缓存，有缓存的话就读取缓存进行渲染

```php
public function fetch($template, $vars = [], $config = [])
    {
。。。
        if (!empty($this->config['cache_id']) && $this->config['display_cache']) {
            // 读取渲染缓存
            $cacheContent = Cache::get($this->config['cache_id']);
            if (false !== $cacheContent) {
                echo $cacheContent;
                return;
            }
        }
    。。。。
            if (!$this->checkCache($cacheFile)) {
                // 缓存无效 重新模板编译
                $content = file_get_contents($template);
                $this->compiler($content, $cacheFile);
            }
            // 页面缓存
            ob_start();
            ob_implicit_flush(0);
            // 读取编译存储
            $this->storage->read($cacheFile, $this->data);
            // 获取并清空缓存
            $content = ob_get_clean();
            if (!empty($this->config['cache_id']) && $this->config['display_cache']) {
                // 缓存页面输出
                Cache::set($this->config['cache_id'], $content, $this->config['cache_time']);
            }
            echo $content;
        }
    }
```



# 视图输出

### 简单的视图工作

```php
#所使用的模板文件为index.html
<?php
namespace app\index\controller;
use think\View;
class Index{
    public function index(){
        $view=View::instance();
        $view->assign('test','你好');				//将你好赋值给模板中的$test
        return $view->fetch('Index/index');
    }
}
```

```html
<html>
    <body>
        输出变量test：{$test}		//$view->assign('test','你好');对应这里
    </body>
</html>
```

### 模板变量赋值

> 使用View::assign('变量名','value')方法进行赋值

```php
 public function assign($name, $value = '')
    {
        if (is_array($name)) {
            $this->data = array_merge($this->data, $name);
        } else {
            $this->data[$name] = $value;
        }
        return $this;
    }
```

### 模板变量转义

> 转义后才会进入正则表达式进行替换

```php
  private function stripPreg($str)
    {
        return str_replace(
            ['{', '}', '(', ')', '|', '[', ']', '-', '+', '*', '.', '^', '?'],
            ['\{', '\}', '\(', '\)', '\|', '\[', '\]', '\-', '\+', '\*', '\.', '\^', '\?'],
            $str);
    }
```

### 模板变量解析

> 赋值完毕后，还得替换

```php
private function parseTag(&$content){
		//获取匹配{$xxx}的正则
        $regex = $this->getRegex('tag');
        //对匹配到文本进行加工处理
        if (preg_match_all($regex, $content, $matches, PREG_SET_ORDER)) {
        ...
        }
}
```

# 缓存

> TP中支持 磁盘缓存 内存缓存 两种

> 在thinkphp/library/cache/driver目录下保存了TP框架不同的缓存驱动。所以TP中所有的缓存驱动都需要继承自think\cache\Driver类。

> 缓存的本质为对不同的缓存媒体的增删改查。比如文件缓存类think\cache\driver\File，该类提供了数据的写入读取，文件的创建删除功能。其中数据写入到文件是通过序列化实现的。而内存缓存比如redis，也只是提供对redis的增删改查。

## 文件缓存

> 文件缓存的本质就是将序列化数据写入到文件中

```php
设置缓存：Cache::set($name, $value, $expire = null)
获取缓存：Cache::get($name, $default = null)
删除缓存：Cache::delete($name)
判断缓存是否存在：Cache::has($name)
自增缓存：Cache::inc($name, $step = 1)
自减缓存：Cache::dec($name, $step = 1)
```



## 内存缓存

> 内存缓存的本质就是 使用 Predis 扩展与 Redis 服务器进行通信，并通过 Predis 提供的方法来实现 Redis 缓存的读写操作

```php
设置缓存：Cache::store('redis')->set($name, $value, $expire = null)
获取缓存：Cache::store('redis')->get($name, $default = null)
删除缓存：Cache::store('redis')->delete($name)
判断缓存是否存在：Cache::store('redis')->has($name)
```





# 安全过滤

> 使用Request::filter和Request::getFilter获取或设置过滤器
>
> 在Request::input中会调用filterValue进行过滤器过滤，调用Request::typecast进行强制类型转换
>
> Request::filterValue会调用过滤器进行过滤，最后会调用表达式过滤器Request::filterExp进行过滤表达式

### 处理链

```php
Request::各种封装类()->Request::input()->
    Request::filterValue()->Request::filteExp
    ->Request::typeCast()
```

## 过滤器

### strip_sql

> 如果是数组的话，使用strip_sql原生函数进行处理
>
> 如果非数组的话，使用preg_replace进行替换处理preg_replace($pattern_arr, $replace_arr, $string);

```php
    function strip_sql($string)
    {
        $pattern_arr = array(
            "/(\s+)union(\s+)/i",
            //一个或多个空白字符union一个或多个空白字符
            "/\bselect\b/i",
            //边界select边界
            "/\bupdate\b/i",
            
            "/\bdelete\b/i",
            "/\boutfile\b/i",
            // "/\bor\b/i",
            "/\bchar\b/i",
            "/\bconcat\b/i",
            "/\btruncate\b/i",
            "/\bdrop\b/i",
            "/\binsert\b/i",
            "/\brevoke\b/i",
            "/\bgrant\b/i",
            "/\breplace\b/i",
            // "/\balert\b/i",
            "/\brename\b/i",
            // "/\bmaster\b/i",
            "/\bdeclare\b/i",
            // "/\bsource\b/i",
            // "/\bload\b/i",
            // "/\bcall\b/i",
            "/\bexec\b/i",
            "/\bdelimiter\b/i",
            "/\bphar\b\:/i",
            "/\bphar\b/i",
            "/\@(\s*)\beval\b/i",
            "/\beval\b/i",
            "/\bonerror\b/i",
            "/\bscript\b/i",
        );
        $replace_arr = array(
            'ｕｎｉｏｎ',
            'ｓｅｌｅｃｔ',
            'ｕｐｄａｔｅ',
            'ｄｅｌｅｔｅ',
            'ｏｕｔｆｉｌｅ',
            // 'ｏｒ',
            'ｃｈａｒ',
            'ｃｏｎｃａｔ',
            'ｔｒｕｎｃａｔｅ',
            'ｄｒｏｐ',
            'ｉｎｓｅｒｔ',
            'ｒｅｖｏｋｅ',
            'ｇｒａｎｔ',
            'ｒｅｐｌａｃｅ',
            // 'ａｌｅｒｔ',
            'ｒｅｎａｍｅ',
            // 'ｍａｓｔｅｒ',
            'ｄｅｃｌａｒｅ',
            // 'ｓｏｕｒｃｅ',
            // 'ｌｏａｄ',
            // 'ｃａｌｌ',
            'ｅｘｅｃ',
            'ｄｅｌｉｍｉｔｅｒ',
            'ｐｈａｒ',
            'ｐｈａｒ',
            '＠ｅｖａｌ',
            'ｅｖａｌ',
            'ｏｎｅｒｒｏｒ',
            'ｓｃｒｉｐｔ',
        );

        return is_array($string) ? array_map('strip_sql', $string) : preg_replace($pattern_arr, $replace_arr, $string);
    }
```

### htmlspecialchars

> 原生的过滤函数
