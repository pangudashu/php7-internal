## 2.5 常量
常量是一个简单值的标识符（名字）。如同其名称所暗示的，在脚本执行期间该值不能改变。常量默认为大小写敏感。通常常量标识符总是大写的。

常量名和其它任何 PHP 标签遵循同样的命名规则。合法的常量名以字母或下划线开始，后面跟着任何字母，数字或下划线。

PHP中的常量通过`define()`函数定义：
```php
define('CONST_VAR_1', 1234);
```
### 2.5.1 常量的存储
在内核中常量存储在`EG(zend_constants)`哈希表中，访问时也是根据常量名直接到哈希表中查找，其实现比较简单。

常量的数据结构：
```c
typedef struct _zend_constant {
    zval value;   //常量值
    zend_string *name; //常量名
    int flags;  //常量标识位
    int module_number; //所属扩展、模块
} zend_constant;
```
常量的几个属性都比较直观，这里只介绍下flags，它的值可以是以下三个中任意组合：
```c
#define CONST_CS                (1<<0)  //大小写敏感
#define CONST_PERSISTENT        (1<<1)  //持久化的
#define CONST_CT_SUBST          (1<<2)  //允许编译时替换
```
介绍下三种flag代表的含义：
* __CONST_CS:__ 大小写敏感，默认是开启的，用户通过define()定义的始终是区分大小写的，通过扩展定义的可以自由选择
* __CONST_PERSISTENT:__ 持久化的，只有通过扩展、内核定义的才支持，这种常量不会在request结束时清理掉
* __CONST_CT_SUBST:__ 允许编译时替换，编译时如果发现有地方在读取常量的值，那么编译器会尝试直接替换为常量值，而不是在执行时再去读取，目前这个flag只有TRUE、FALSE、NULL三个常量在使用

### 2.5.2 常量的销毁
非持久化常量在request请求结束时销毁，具体销毁操作在：`php_request_shutdown()->zend_deactivate()->shutdown_executor()->clean_non_persistent_constants()`。
```c
void clean_non_persistent_constants(void)
{
    if (EG(full_tables_cleanup)) {
        zend_hash_apply(EG(zend_constants), clean_non_persistent_constant_full);
    } else {
        zend_hash_reverse_apply(EG(zend_constants), clean_non_persistent_constant);
    }
}
```
然后从哈希表末尾开始向前遍历EG(zend_constants)，将非持久化常量删除，直到碰到第一个持久化常量时，停止遍历，正常情况下所有通过扩展定义的常量一定是在PHP中通过define定义之前，当然也并非绝对，这里只是说在所有常量均是在MINT阶段定义的情况。

持久化常量是在`php_module_shutdown()`阶段销毁的，具体过程与上面类似。
