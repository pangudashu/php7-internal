## 7.5 运行时配置

### 7.5.1 全局变量(资源)
使用C语言开发程序时经常会使用全局变量进行数据存储，这就涉及前面已经介绍过的一个问题：线程安全，PHP设计了TSRM（即：线程安全资源管理器）用于解决这个问题，内核中频繁使用到的EG、CG等都是根据是否开启ZTS封装的宏，同样的，在扩展中也需要必须按照TSRM的规范定义全局变量，除非你的扩展不支持多线程的环境。

PHP为扩展的全局变量提供了一种存储方式：每个扩展将自己所有的全局变量统一定义在一个结构体中，然后将这个结构体注册到TSRM中，这样扩展就可以像使用EG、CG那样访问这个结构体。

这个结构体的定义通过`ZEND_BEGIN_MODULE_GLOBALS(extension_name)`、`ZEND_END_MODULE_GLOBALS(extension_name)`两个宏完成，这两个宏必须成对出现，中间定义扩展需要的全局变量即可。
```c
ZEND_BEGIN_MODULE_GLOBALS(mytest)
	zend_long	open_cache;
	HashTable	class_table;
ZEND_END_MODULE_GLOBALS(mytest)
```
展开后实际就是个普通的struct：
```c
typedef struct _zend_mytest_globals {
	zend_long   open_cache;
	HashTable   class_table;
}zend_mytest_globals;
```
接着创建一个此结构体的全局变量，这时候就会涉及ZTS了，如果未开启线程安全直接创建普通的全局变量即可，如果开启线程安全了则需要向TSRM注册，得到一个唯一的资源id，这个操作也由专门的宏来完成：`ZEND_DECLARE_MODULE_GLOBALS(extension_name)`，展开后：
```c
//ZTS：此时只是定义资源id，并没有向TSRM注册
ts_rsrc_id mytest_globals_id;

//非ZTS
zend_mytest_globals mytest_globals;
```
最后需要定义一个像EG、CG那样的宏用于访问扩展的全局资源结构体，这一步将使用`ZEND_MODULE_GLOBALS_ACCESSOR()`宏完成：
```c
#define MYTEST_G(v) ZEND_MODULE_GLOBALS_ACCESSOR(mytest, v)
```
看起来是不是跟EG、CG的定义非常像？这个宏展开后：
```c
//ZTS
#define MYTEST_G(v) ZEND_TSRMG(mytest_globals_id, zend_##module_name##_globals *, v)

//非ZTS
#define MYTEST_G(v) (mytest_globals.v)
```
接下来就可以在扩展中通过：MYTEST_G(opene_cache)、MYTEST_G(class_table)对结构体成员进行读写了。通常会把这个全局资源结构体及结构体的访问宏定义在头文件中，然后把全局变量的声明放到源文件中：
```c
//php_mytest.h
#define MYTEST_G(v) ZEND_MODULE_GLOBALS_ACCESSOR(mytest, v)

ZEND_BEGIN_MODULE_GLOBALS(mytest)
	zend_long   open_cache;
	HashTable   class_table;
ZEND_END_MODULE_GLOBALS(mytest)

//mytest.c
ZEND_DECLARE_MODULE_GLOBALS(mytest)
```
> 在一个扩展中并不是只能定义一个全局变量结构，数目是不限制的。

### 7.5.2 php.ini配置
php.ini是PHP主要的配置文件，解析时PHP将在这些地方依次查找该文件：当前工作目录、环境变量PHPRC指定目录、编译时指定的路径，在命令行模式下，php.ini的查找路径可以用`-c`参数替代。

该文件的语法非常简单：`配置标识符 = 值`。空白字符和用分号';'开始的行被忽略，[xxx]行也被忽略；配置标识符大写敏感，通常会用'.'区分不同的节；值可以是数字、字符串、PHP常量、位运算表达式。

关于php.ini的解析过程本节不作介绍，只从应用的角度介绍如何在一个扩展中获取一个配置项，通常会把php.ini的配置映射到一个变量，从而在使用时直接读取那个变量，也就是把所有的配置转化为了C语言中的变量，扩展中一般会把php.ini配置映射到上一节介绍的全局变量(资源)，要想实现这个转化需要在扩展中为每一项配置设置映射规则：
```c
PHP_INI_BEGIN()
	//每一项配置规则
	...
PHP_INI_END();
```
这两个宏实际只是把各配置规则组成一个数组，配置规则通过`STD_PHP_INI_ENTRY()`设置：
```c
STD_PHP_INI_ENTRY(name,default_value,modifiable,on_modify,property_name,struct_type,struct_ptr)
```
* __name:__ php.ini中的配置标识符
* __default_value:__ 默认值，注意不管转化后是什么类型，这里必须设置为字符串
* __modifiable:__
* __on_modify:__ 函数指针，用于指定发现这个配置后赋值处理的函数，默认提供了5个：OnUpdateBool、OnUpdateLong、OnUpdateLongGEZero、OnUpdateReal、OnUpdateString、OnUpdateStringUnempty，如果满足不了需求可以自定义
* __property_name:__ 要映射到的结构struct_type中的成员
* __struct_type:__ 映射结构的类型
* __struct_ptr:__ 映射结构的变量地址，发现配置后会

这个宏展开后生成一个`zend_ini_entry_def`结构：
```c
typedef struct _zend_ini_entry_def {
	const char *name;
	int (*on_modify)(zend_ini_entry *entry, zend_string *new_value, void *mh_arg1, void *mh_arg2, void *mh_arg3, int stage);
	void *mh_arg1; //映射成员所在结构体的偏移:offsetof(type, member-designator)取到
	void *mh_arg2; //要映射到结构的地址
	void *mh_arg3;
	const char *value;//默认值
	void (*displayer)(zend_ini_entry *ini_entry, int type);
	int modifiable;

	uint name_length;
	uint value_length;
} zend_ini_entry_def;
```
比如将php.ini中的`mytest.opene_cache`值映射到`MYTEST_G()`结构中的open_cache，类型为zend_long，默认值109，则可以这么定义：
```c
PHP_INI_BEGIN()
	STD_PHP_INI_ENTRY("mytest.open_cache", "109", PHP_INI_ALL, OnUpdateLong, open_cache, zend_mytest_globals, mytest_globals)
PHP_INI_END();
```
property_name设置的是要映射到的结构成员`mytest_globals->open_cache`，zend_mytest_globals、mytest_globals都是宏展开后的实际值，前者是结构体类型，后者是具体分配的变量，

上面的定义展开后：
```c
static const zend_ini_entry_def ini_entries[] = {
	{
		"mytest.open_cache", 
		OnUpdateLong, 
		(void *) XtOffsetOf(zend_mytest_globals, open_cache), //获取成员在结构体中的内存偏移
		(void*)&mytest_globals,
		NULL,
		"109",
		NULL,
		PHP_INI_ALL,
		sizeof("mytest.open_cache")-1,
		sizeof("109")-1
	},
	{ NULL, NULL, NULL, NULL, NULL, NULL, NULL, 0, 0, 0}
}
```
> `XtOffsetOf()`这个宏在linux环境下展开就是`offsetof()`，用来获取一个结构体成员的offset，比如：
>
> #include <stdio.h>    
> #include <stddef.h>   
>
> typedef struct{   
> 	  int     id;   
>     char    *name;   
> }my_struct;
> 
> int main(void)   
> {    
>     printf("%d\n", (void*)offsetof(my_struct, name));   
>	  return 0;   
> }
>
> 通过这个offset及结构体指针就可以读取这个成员：`(char*)my_sutct + offset`，等价于`my_sutct->name`。


