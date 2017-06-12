## 7.9 面向对象
### 7.9.1 定义内部类
在扩展中定义一个内部类的方式与函数类似，函数最终注册到EG(function_table)，而类则最终注册到EG(class_table)符号表中，注册的过程首先是为类创建一个zend_class_entry结构，然后把这个结构插入EG(class_table)，当然这个过程不需要我们手动操作，PHP提供了现成的方法和宏帮我们对zend_class_entry进行初始化以及注册。通常情况下会把内部类的注册放到module startup阶段，也就是定义在扩展的`PHP_MINIT_FUNCTION()`中，一个简单的类的注册只需要以下几行：
```c
PHP_MINIT_FUNCTION(mytest)
{
    //分配一个zend_class_entry，这个结构只在注册时使用，所以分配在栈上即可
    zend_class_entry ce;
    //对zend_class_entry进行初始化
    INIT_CLASS_ENTRY(ce, "MyClass", NULL);
    //注册
    zend_register_internal_class(&ce);
}
```
这样就成功定义了一个内部类，类名为"MyClass"，只是这个类还没有任何的成员属性、成员方法，定义完成后重新编译、安装扩展，然后在PHP脚本中实例化这个类：
```php
$obj = new MyClass();

var_dump($obj);
```
结果将输出：
```
object(MyClass)#1 (0) {
}
```
注册时传入的zend_class_entry并不是最终插入class_table符号表的结构，zend_register_internal_class()中会重新分配，所以注册时的这个结构分配在栈上即可，此结构的成员不需要手动定义，PHP提供了宏供扩展使用，扩展只需要提供类的主要信息即可，常用的两个宏：
```c
/**
 * 初始化zend_class_entry
 * class_container：zend_class_entry地址
 * class_name：类名
 * functions：成员方法数组
 */
#define INIT_CLASS_ENTRY(class_container, class_name, functions) \
    INIT_OVERLOADED_CLASS_ENTRY(class_container, class_name, functions, NULL, NULL, NULL)

/**
 * 初始化zend_class_entry，带namespace
 * class_container：zend_class_entry地址
 * ns：命名空间
 * class_name：类名
 * functions：成员方法数组
 */
#define INIT_NS_CLASS_ENTRY(class_container, ns, class_name, functions) \
    INIT_CLASS_ENTRY(class_container, ZEND_NS_NAME(ns, class_name), functions)
```

### 7.9.2 定义成员属性

### 7.9.3 定义成员方法

### 7.9.4 定义常量

### 7.9.5 类的实例化
