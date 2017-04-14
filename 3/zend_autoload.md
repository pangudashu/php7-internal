### 3.4.7 类的自动加载
在实际使用中，通常会把一个类定义在一个文件中，然后使用时include加载进来，这样就带来一个问题：在每个文件的头部都需要包含一个长长的include列表，而且当文件名称修改时也需要把每个引用的地方都改一遍，另外前面我们也介绍过，原则上父类需要在子类定义之前定义，当存在大量类时很难得到保证，因此PHP提供了一种类的自动加载机制，当使用未被定义的类时自动调用类加载器将类加载进来，方便类的同一管理。

PHP中提供了两种方式实现自动加载：`__autoload()`、`spl_autoload_register()`，两种方式差别不大，我们先介绍下他们的用法再分析PHP中的实现原理。

__(1)__autoload():__

用户自定义一个`__autoload()`函数即可，参数是类名，当实例化一个类是如果没有找到这个类则会调用此函数，所以我们就可以在这个函数中进行类的加载，比如：
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

`spl_autoload_register()`提供了更加灵活的注册方式，可以支持任意数量的加载器，比如第三方库加载规则不可能保持一致，这样就可以通过此函数注册自己的加载器了。

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

___


