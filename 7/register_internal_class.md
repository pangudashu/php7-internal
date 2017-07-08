### 7.9.1.1 注册一个内部类
作为一个PHP工程师我们平时开发见到比较多的词应该就是 `class`, 那么在我们平时通过PHP脚本创建的类，怎样通过ZEND API去实现一个PHP内部类呢? 本章将会向你们讲述如何创建一个简单的内部类。

### 需要使用到的ZEND API

| API  
| --- |
|[zend_class_entry]()| 
|[zend_register_internal_class()]()|
|[INIT_CLASS_ENTRY()]()|
|[PHP_METHOD()]()|

* 如果你对如何创建一个PHP扩展骨架还不太清楚建议你阅读鸟哥的 [ 用C/C++扩展你的PHP](http://www.laruence.com/2009/04/28/719.html)
* 上面几个API在本书 `类`的章节都有描述， 如果对这几个核心API还不是很了解，建议重读本书关于`类`的几个章节

### 创建一个简单的内部类
假设我们创建一个叫SimpleClass的类用PHP脚本实现是这样的:

```
<?php 
	
namespace Example;
	
class SimpleClass {
	public function helloWorld()
	{
		echo "helloWorld\n";
	}
}
?>	
```

如果我们想通过ZEND_API用C语言去实现一个内部类，那我们就需要做到以下几个步骤:
* 声明一个类
* 声明一个方法
* 声明一个类容器
* 将方法装载到类容器里
* 通过ZEND_API将类注册为内部类

用扩展实现是这样的: 
* ***simple_class.h***

```
#ifndef SIMPLE_CLASS_H
#define SIMPLE_CLASS_H 1
PHP_METHOD(SimpleClass, helloWorld);
#endif

```
* ***simple_class.c***

```
#ifdef HAVE_CONFIG_H
#include "config.h"
#endif

#include "php.h"
#include "simple_class.h"
/* 声明类方法 */
PHP_METHOD(SimpleClass, helloWorld)
{
    php_printf("hello world!!!\n");
}

const zend_function_entry simple_class_methods[] = {
    PHP_ME(SimpleClass, helloWorld, NULL, ZEND_ACC_PUBLIC)
    PHP_FE_END
};

ZEND_MINIT_FUNCTION(SimpleClass)
{
	/* 声明类容器 */
    zend_class_entry simple_class_container_ce; 
    /* 将类方法注入到类容器中 */
    INIT_CLASS_ENTRY(simple_class_container_ce, "Example\\SimpleClass", simple_class_methods);
    /* 将类容器注册为内部类, 这个APi将会返回一个类型为zend_class_entry的指针 */
    zend_register_internal_class(&simple_class_container_ce);
    return SUCCESS;
}
```

### 本章节源码
* [SimpleClass](https://github.com/motecshine/php-ext-example/tree/master/class) 

