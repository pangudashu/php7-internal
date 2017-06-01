## 7.8 常量
常量的具体实现前面章节已经介绍过，这里不再重复。PHP提供了很多用于常量注册的宏，可以在扩展的`PHP_MINIT_FUNCTION()`中定义：
```c
//注册NULL常量
#define REGISTER_NULL_CONSTANT(name, flags)  \
    zend_register_null_constant((name), sizeof(name)-1, (flags), module_number)

//注册bool常量
#define REGISTER_BOOL_CONSTANT(name, bval, flags)  \
    zend_register_bool_constant((name), sizeof(name)-1, (bval), (flags), module_number)

//注册整形常量
#define REGISTER_LONG_CONSTANT(name, lval, flags)  \
    zend_register_long_constant((name), sizeof(name)-1, (lval), (flags), module_number)

//注册浮点型常量
#define REGISTER_DOUBLE_CONSTANT(name, dval, flags)  \
    zend_register_double_constant((name), sizeof(name)-1, (dval), (flags), module_number)

//注册字符串常量，str类型为char*
#define REGISTER_STRING_CONSTANT(name, str, flags)  \
    zend_register_string_constant((name), sizeof(name)-1, (str), (flags), module_number)

//注册字符串常量，截取指定长度，str类型为char*
#define REGISTER_STRINGL_CONSTANT(name, str, len, flags)  \
    zend_register_stringl_constant((name), sizeof(name)-1, (str), (len), (flags), module_number)
```
除了上面这些还有`REGISTER_NS_XXX`系列的宏用于带namespace的常量注册，另外如果这些类型不能满足需求，则可以通过`zend_register_constant(zend_constant *c)`注册，比如常量类型为数组。
```c
PHP_MINIT_FUNCTION(mytest)
{   
    ...

    REGISTER_STRING_CONSTANT("MY_CONS_1", "this is a constant", CONST_CS | CONST_PERSISTENT);
}
```
```php
echo MY_CONS_1;
=========[output]=========
this is a constant
```
如果在扩展中需要用到其他扩展或内核定义的常量，则可以通过以下函数获取常量的值：
```c
ZEND_API zval *zend_get_constant(zend_string *name);
ZEND_API zval *zend_get_constant_str(const char *name, size_t name_len);
```
