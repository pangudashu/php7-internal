## 7.2 扩展的实现原理
PHP中扩展通过`zend_module_entry`这个结构来表示，此结构定义了扩展的全部信息：扩展名、扩展版本、扩展提供的函数列表以及PHP四个执行阶段的hook函数等，每一个扩展都需要定义一个此结构的变量，而且这个变量的名称格式必须是：`{module_name}_module_entry`，内核正是通过这个结构获取到扩展提供的功能的。

扩展可以在编译PHP时一起编译(静态编译)，也可以单独编译为动态库，动态库需要加入到php.ini配置中去，然后在`php_module_startup()`阶段把这些动态库加载到PHP中：
```c
int php_module_startup(sapi_module_struct *sf, zend_module_entry *additional_modules, uint num_additional_modules)
{
	...
	//根据php.ini注册扩展
	php_ini_register_extensions();
	zend_startup_modules();

	zend_startup_extensions();
	...
}
```
动态库就是在`php_ini_register_extensions()`这个函数中完成的注册：
```c
//main/php_ini.c
void php_ini_register_extensions(void)
{
	//注册zend扩展
    zend_llist_apply(&extension_lists.engine, php_load_zend_extension_cb);
	//注册php扩展
    zend_llist_apply(&extension_lists.functions, php_load_php_extension_cb);

    zend_llist_destroy(&extension_lists.engine);
    zend_llist_destroy(&extension_lists.functions);
}
```
extension_lists是一个链表，保存着根据`php.ini`中定义的`extension=xxx.so`取到的全部扩展名称，其中engine是zend扩展，functions为php扩展，依次遍历这两个数组然后调用`php_load_php_extension_cb()`或`php_load_zend_extension_cb()`进行各个扩展的加载：
```c
static void php_load_php_extension_cb(void *arg)
{
#ifdef HAVE_LIBDL
    php_load_extension(*((char **) arg), MODULE_PERSISTENT, 0);
#endif
}
```
`HAVE_LIBDL`这个宏根据`dlopen()`函数是否存在设置的：
```sh
#Zend/Zend.m4
AC_DEFUN([LIBZEND_LIBDL_CHECKS],[
AC_CHECK_LIB(dl, dlopen, [LIBS="-ldl $LIBS"])
AC_CHECK_FUNC(dlopen,[AC_DEFINE(HAVE_LIBDL, 1,[ ])])
])
```
接着就是最关键的操作了，`php_load_extension()`：
```c
//ext/standard/dl.c
PHPAPI int php_load_extension(char *filename, int type, int start_now)
{
	void *handle;
	char *libpath;
	zend_module_entry *module_entry;
	zend_module_entry *(*get_module)(void);
    ...
	//调用dlopen打开指定的动态连接库文件：xx.so
    handle = DL_LOAD(libpath); 
    ...
	//调用dlsym获取get_module的函数指针
    get_module = (zend_module_entry *(*)(void)) DL_FETCH_SYMBOL(handle, "get_module"); 
    ...
	//调用扩展的get_module()函数
    module_entry = get_module();
    ...
	//检查扩展使用的zend api是否与当前php版本一致
	if (module_entry->zend_api != ZEND_MODULE_API_NO) {
		DL_UNLOAD(handle);
		return FAILURE;
	}
	...
	module_entry->type = type;
	//为扩展编号
	module_entry->module_number = zend_next_free_module();
	module_entry->handle = handle;

	if ((module_entry = zend_register_module_ex(module_entry)) == NULL) {
		DL_UNLOAD(handle);
		return FAILURE;
	}
	...
}
```
`DL_LOAD()`、`DL_FETCH_SYMBOL()`这两个宏在linux下展开后就是：dlopen()、dlsym()，所以上面过程的实现就比较直观了：

* (1)dlopen()打开so库文件；
* (2)dlsym()获取动态库中`get_module()`函数的地址，`get_module()`是每个扩展都必须提供的一个接口，用于返回扩展`zend_module_entry`结构的地址;
* (3)调用扩展的`get_module()`，获取扩展的`zend_module_entry`结构;
* (4)zend api版本号检查，比如php7的扩展在php5下是无法使用的；
* (5)注册扩展，将扩展添加到`module_registry`中，这是一个全局HashTable，用于全部扩展的zend_module_entry结构；
* (6)如果扩展提供了内部函数则将这些函数注册到EG(function_table)中。

完成扩展的注册后，PHP将在不同的执行阶段依次调用每个扩展注册的当前阶段的hook函数。
