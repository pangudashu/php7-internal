## 2.3 静态变量
PHP中局部变量分配在zend_execute_data结构上，每次执行zend_op_array都会生成一个新的zend_execute_data，局部变量在执行之初分配，然后在执行结束时释放，这是局部变量的生命周期，而局部变量中有一种特殊的类型：静态变量，它们不会在函数执行完后释放，当程序执行离开函数域时静态变量的值被保留下来，下次执行时仍然可以使用之前的值。

PHP中的静态变量通过`static`关键词创建：
```php
function my_func(){
    static $count = 4;
    $count++;
    echo $count,"\n";
}
my_func();
my_func();
my_func();

===========================
5
6
7
```
静态变量既然不会随执行的结束而释放，那么很容易想到它的保存位置：`zend_op_array->static_variables`，这是一个哈希表，所以PHP中的静态变量与普通局部变量不同，它们没有分配在执行空间zend_execute_data上，而是以哈希表的形式保存在zend_op_array中。

静态变量只会初始化一次，注意：它的初始化发生在编译阶段而不是执行阶段。

//：ZEND_FETCH_W、ZEND_ASSIGN_REF，`zend_assign_to_variable_reference()`
