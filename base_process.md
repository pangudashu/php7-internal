# 执行流程

![php_process](img/php.png)

## 扩展加载过程

PHP扩展的结构`zend_module_entry`：
```c
//zend_modules.h
struct _zend_module_entry {
    unsigned short size;
    unsigned int zend_api;
    unsigned char zend_debug;
    unsigned char zts;
    const struct _zend_ini_entry *ini_entry;
    const struct _zend_module_dep *deps;
    const char *name; //扩展名称，不能重复
    const struct _zend_function_entry *functions; //扩展提供的内部函数列表
    int (*module_startup_func)(INIT_FUNC_ARGS); //扩展初始化回调函数，PHP_MINIT_FUNCTION或ZEND_MINIT_FUNCTION定义的函数
    int (*module_shutdown_func)(SHUTDOWN_FUNC_ARGS); //扩展关闭时回调函数
    int (*request_startup_func)(INIT_FUNC_ARGS); //请求开始前回调函数
    int (*request_shutdown_func)(SHUTDOWN_FUNC_ARGS); //请求结束时回调函数
    void (*info_func)(ZEND_MODULE_INFO_FUNC_ARGS); //php_info展示的扩展信息处理函数
    const char *version; //版本
    size_t globals_size;
#ifdef ZTS
    ts_rsrc_id* globals_id_ptr;
#else
    void* globals_ptr;
#endif
    void (*globals_ctor)(void *global);
    void (*globals_dtor)(void *global);
    int (*post_deactivate_func)(void);
    int module_started;
    unsigned char type;
    void *handle;
    int module_number;
    const char *build_id;
};
```
`zend_module_entry`是一个扩展唯一的标识，从这个结构我们可以猜测一个PHP扩展主要都由哪些部分构成：
* __扩展名称__：name，这个是必须的，每个扩展需要有个名字
* __两个阶段的四个hook钩子函数__：module_startup_func、module_shutdown_func、request_startup_func、request_shutdown_func，这些函数分别通过：PHP_MINIT_FUNCTION、PHP_MSHUTDOWN_FUNCTION、PHP_RINIT_FUNCTION、PHP_RSHUTDOWN_FUNCTION几个宏定义，这几个函数不是必须的，可以设为null
* __函数数组__：functions，这个是指扩展中定义的PHP内部函数，是个数组，内部函数通过宏`PHP_FUNCTION`定义，这个也不是必须的可以设为null

总的来看一个扩展主要就包括上面三个部分，其中`zend_module_entry`是最重要的一个结构，我们需要定义一个这样的全局变量：
```c
zend_module_entry {mudule_name}_module_entry = {
    ...
}
```
而且这个变量的名称格式必须是：__{mudule_name}_module_entry__。

扩展可以在编译PHP时一起编译，也可以后期编译然后动态加载，下面看下动态扩展的加载过程：

首先调用`php_ini_register_extensions`：
```c
//main/php_ini.c
void php_ini_register_extensions(void)
{
    zend_llist_apply(&extension_lists.engine, php_load_zend_extension_cb);
    zend_llist_apply(&extension_lists.functions, php_load_php_extension_cb);

    zend_llist_destroy(&extension_lists.engine);
    zend_llist_destroy(&extension_lists.functions);
}
```
`extension_lists`记录的是根据`php.ini`中定义的`extension=xxx.so`取到的全部扩展名称，然后遍历这个数组依次调用`php_load_php_extension_cb`加载扩展(php_load_zend_extension_cb是zend扩展加载时的方法)：
```c
static void php_load_php_extension_cb(void *arg)
{
#ifdef HAVE_LIBDL
    php_load_extension(*((char **) arg), MODULE_PERSISTENT, 0);
#endif
}
```
`php_load_extension`：
```c
//ext/standard/dl.c #90
PHPAPI int php_load_extension(char *filename, int type, int start_now)
{
    ...

    zend_module_entry *module_entry;
    zend_module_entry *(*get_module)(void);
    ...

    handle = DL_LOAD(libpath); //调用dlopen打开指定的动态连接库文件：xx.so
    ...

    get_module = (zend_module_entry *(*)(void)) DL_FETCH_SYMBOL(handle, "get_module"); //调用dlsym获取get_module的函数指针
    ...

    module_entry = get_module();
    ...
}
```
上面的过程就是普通的动态链接库的用法了：通过`dlopen`打开库文件，通过`dlsym`获取指定函数、变量，最终得到的`module_entry`就是我们在扩展中定义的`zend_module_entry`结构，拿到这个结构接下来的过程就比较简单了，主要就是检查版本是否匹配可用、调用`zend_register_module_ex`将扩展加到`module_registry`（全部扩展的哈希表）、注册扩展提供的函数、调用扩展初始化回调函数。

还有一个地方值得注意的是调用dlsym获取get_module的函数指针的过程，这说明每个扩展中都需要定义一个__get_module()__函数，这个函数返回了我们定义的`zend_module_entry`结构指针，实际我们可以根据`ZEND_GET_MODULE(module_name)`这个宏完成这个函数的定义：
```c
//zend_API.h
#define ZEND_GET_MODULE(name) \
    BEGIN_EXTERN_C()\
    ZEND_DLEXPORT zend_module_entry *get_module(void) { return &name##_module_entry; //这个就是我们定义的扩展结构的全局变量 }\
    END_EXTERN_C()
```

