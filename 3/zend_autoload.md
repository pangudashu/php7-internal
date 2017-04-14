### 3.4.7 类的自动加载
在实际使用中，通常会把一个类定义在一个文件中，然后使用时include加载进来，这样就带来一个问题：在每个文件的头部都需要包含一个长长的include列表，而且当文件名称修改时也需要把每个引用的地方都改一遍，另外前面我们也介绍过，原则上父类需要在子类定义之前定义，当存在大量类时很难得到保证，因此PHP提供了一种类的自动加载机制，当使用未被定义的类时自动调用类加载器将类加载进来，方便类的同一管理。

在内核实现上类的自动加载实际就是定义了一个钩子函数，实例化类时如果在EG(class_table)中没有找到对应的类则会调用这个钩子函数，调用完以后再重新查找一次。这个钩子函数保存在EG(autoload_func)中。

PHP中提供了两种方式实现自动加载：`__autoload()`、`spl_autoload_register()`。

***(1)__autoload():***

这种方式比较简单，用户自定义一个`__autoload()`函数即可，参数是类名，当实例化一个类是如果没有找到这个类则会查找用户是否定义了`__autoload()`函数，如果定义了则调用此函数，比如：
```php
//文件1：my_class.php
<?php
class my_class {
    public $id = 123;
}

//文件2：b.php
<?php
function __autoload($class_name){
    //do something...
    include $class_name . '.php';
}

$obj = new my_class();
var_dump($obj);
```

__(2)spl_autoload_register():__

相比`__autoload()`只能定义一个加载器，`spl_autoload_register()`提供了更加灵活的注册方式，可以支持任意数量的加载器，比如第三方库加载规则不可能保持一致，这样就可以通过此函数注册自己的加载器了，在实现上spl创建了一个队列来保存用户注册的加载器，然后定义了一个spl_autoload函数到EG(autoload_func)，当找不到类时内核回调spl_autoload，这个函数再依次调用用户注册的加载器，没调用一个重新检查下查找的类是否在EG(class_table)中已经注册，仍找不到的话继续调用下一个加载器，直到类成功注册为止。

```c
bool spl_autoload_register ([ callable $autoload_function [, bool $throw = true [, bool $prepend = false ]]] )
```
参数`$autoload_function`为加载器，可以是函数名，第2个参数`$throw`用于设置autoload_function 无法成功注册时， spl_autoload_register()是否抛出异常，最后一个参数如果为true时spl_autoload_register() 会添加函数到队列之首，而不是队列尾部。

```php
function autoload_one($class_name){
    echo "autoload_one->", $class_name, "\n";
}

function autoload_two($class_name){
    echo "autoload_two->", $class_name, "\n";
}

spl_autoload_register("autoload_one");
spl_autoload_register("autoload_two");

$obj = new my_class();
var_dump($obj);
```
这个例子执行时就会将autoload_one()、autoload_two()都调一遍，假如第一个函数就成功注册了my_class类则不会再调后面的加载器。

内核查找类通过`zend_lookup_class_ex()`完成，我们简单看下其处理过程。
```c
//file: zend_execute_API.c
ZEND_API zend_class_entry *zend_lookup_class_ex(zend_string *name, const zval *key, int use_autoload)
{
    ...
    //从EG(class_table)符号表找类的zend_class_entry，如果找到说明类已经编译，直接返回
    ce = zend_hash_find_ptr(EG(class_table), lc_name);
    if (ce) {
        if (!key) {
            zend_string_release(lc_name);
        }
        return ce;
    }
    ...
    //如果没有通过spl注册则看下是否定义了__autoload()
    if (!EG(autoload_func)) {
        zend_function *func = zend_hash_str_find_ptr(EG(function_table), "__autoload", sizeof("__autoload") - 1);
        if (func) {
            EG(autoload_func) = func;
        } else {
            return NULL;
        }
    }
    ...
    fcall_cache.function_handler = EG(autoload_func);
    ...
    //调用EG(autoload_func)函数，然后再查一次EG(class_table)
    if ((zend_call_function(&fcall_info, &fcall_cache) == SUCCESS) && !EG(exception)) {
        ce = zend_hash_find_ptr(EG(class_table), lc_name);
    }
    ...
}
```
SPL的具体实现比较简单，这里不再介绍。
