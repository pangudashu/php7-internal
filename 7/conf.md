## 7.5 扩展配置

### 7.5.1 全局变量(资源)
使用C语言开发程序时经常会使用全局变量进行数据存储，这就涉及前面已经介绍过的一个问题：线程安全，PHP设计了TSRM（即：线程安全资源管理器）用于解决这个问题，内核中频繁使用到的EG、CG等都是根据是否开启ZTS封装的宏，同样的，在扩展中也需要必须按照TSRM的规范定义全局变量，除非你的扩展不支持多线程的环境。

PHP为扩展的全局变量提供了一种存储方式：每个扩展将自己所有的全局变量统一定义在一个结构体中，然后将这个结构体注册到TSRM中，这样扩展就可以像使用EG、CG那样访问这个结构体。

这个结构体的定义通过`ZEND_BEGIN_MODULE_GLOBALS(extension_name)`、`ZEND_END_MODULE_GLOBALS(extension_name)`两个宏完成，这两个宏必须成对出现，中间定义扩展需要的全局变量即可。
```c
ZEND_BEGIN_MODULE_GLOBALS(mytest)
    zend_long	opene_cache;
	HashTable	class_table;
ZEND_END_MODULE_GLOBALS(mytest)
```
展开后实际就是个普通的struct：
```c
typedef struct _zend_mytest_globals {
	zend_long   opene_cache;
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
	zend_long   opene_cache;
	HashTable   class_table;
ZEND_END_MODULE_GLOBALS(mytest)

//mytest.c
ZEND_DECLARE_MODULE_GLOBALS(mytest)
```
### 7.5.2 php.ini配置
