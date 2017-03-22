## 3.2 函数实现
函数，通俗的讲就是一组操作的集合，给予特定的输入将对应特定的输出。

### 3.2.1 用户自定义函数的实现
用户自定义函数是指我们在PHP脚本通过function定义的函数：
```php
function my_func(){
    ...
}
```
汇编中函数对应的是一组独立的汇编指令，然后通过call指令实现函数的调用，前面已经说过PHP编译的结果是opcode数组，与汇编指令对应，PHP用户自定义函数的实现就是将函数编译为独立的opcode数组，调用时分配独立的执行栈依次执行opcode，所以自定义函数对于zend而言并没有什么特别之处，只是将opcode进行了打包封装，实际PHP脚本中函数之外的指令整个可以认为是一个函数(或者理解为main函数更直观)。

```php
/* function main(){ */

$a = 123;
echo $a;

/* } */
```
#### 3.2.1.1 函数的存储结构
下面具体看下PHP中函数的结构：

```c
typedef union  _zend_function        zend_function;

//zend_compile.h
union _zend_function {
    zend_uchar type;    /* MUST be the first element of this struct! */

    struct {
        zend_uchar type;  /* never used */
        zend_uchar arg_flags[3]; /* bitset of arg_info.pass_by_reference */
        uint32_t fn_flags;
        zend_string *function_name;
        zend_class_entry *scope; //成员方法所属类，面向对象实现中用到
        union _zend_function *prototype;
        uint32_t num_args; //参数数量
        uint32_t required_num_args; //必传参数数量
        zend_arg_info *arg_info; //参数信息
    } common;

    zend_op_array op_array; //函数实际编译为普通的zend_op_array
    zend_internal_function internal_function;
};
```
这是一个union，因为PHP中函数除了用户自定义函数还有一种：内部函数，内部函数是通过扩展或者内核提供的C函数，比如time、array系列等等，内部函数稍后再作分析。

内部函数主要用到`internal_function`，而用户自定义函数编译完就是一个普通的opcode数组，用的是`op_array`（注意：op_array、internal_function是嵌入的两个结构，而不是一个单独的指针），除了这两个上面还有一个`type`跟`common`，这俩是做什么用的呢？

经过比较发现`zend_op_array`与`zend_internal_function`结构的起始位置都有`common`中的几个成员，如果你对C的内存比较了解应该会马上想到它们的用法，实际`common`可以看作是`op_array`、`internal_function`的header，不管是什么哪种函数都可以通过`zend_function.common.xx`快速访问到`zend_function.zend_op_array.xx`及`zend_function.zend_internal_function.xx`，下面几个，`type`同理，可以直接通过`zend_function.type`取到`zend_function.op_array.type`及`zend_function.internal_function.type`。

![php function](../img/php_function.jpg)

函数是在编译阶段确定的，那么它们存在哪呢？

在PHP脚本的生命周期中有一个非常重要的值`executor_globals`(非ZTS下)，类型是`struct _zend_executor_globals`，它记录着PHP生命周期中所有的数据，如果你写过PHP扩展一定用到过`EG`这个宏，这个宏实际就是对`executor_globals`的操作:`define EG(v) (executor_globals.v)`

`EG(function_table)`是一个哈希表，记录的就是PHP中所有的函数。

PHP在编译阶段将用户自定义的函数编译为独立的opcodes，保存在`EG(function_table)`中，调用时重新分配新的zend_execute_data(相当于运行栈)，然后执行函数的opcodes，调用完再还原到旧的`zend_execute_data`，继续执行，关于zend引擎execute阶段后面会详细分析。

#### 3.2.1.2 函数参数
函数参数在内核实现上与函数内的局部变量实际是一样的，上一篇我们介绍编译的时候提供局部变量会有一个单独的 __编号__ ，而函数的参数与之相同，参数名称也在zend_op_array.vars中，编号首先是从参数开始的，所以按照参数顺序其编号依次为0、1、2...(转化为相对内存偏移量就是96、112、128...)，然后函数调用时首先会在调用位置将参数的value复制到各参数各自的位置，详细的传参过程我们在执行一篇再作说明。

另外参数还有其它的信息，比如默认值、引用传递，这些信息通过`zend_arg_info`结构记录：
```c
typedef struct _zend_arg_info {
    zend_string *name; //函数名
    zend_string *class_name;
    zend_uchar type_hint; //显式声明的参数类型，比如(array $param_1)
    zend_uchar pass_by_reference; //是否引用传参，参数前加&的这个值就是1
    zend_bool allow_null; //是否允许为空
    zend_bool is_variadic; //是否为可变参数，即...用法，与golang的用法相同，5.6以上新增的一个用法
} zend_arg_info;
```
每个参数都有一个上面的结构，所有参数的结构保存在`zend_op_array.arg_info`数组中。

#### 3.2.1.3 函数的编译

### 3.2.2 内部函数
上一节已经提过，内部函数指的是由内核、扩展提供的C语言编写的function，这类函数不需要经历opcode的编译过程，所以效率上要高于PHP用户自定义的函数，调用时与普通的C程序没有差异。

Zend引擎中定义了很多内部函数供用户在PHP中使用，比如：define、defined、strlen、method_exists、class_exists、function_exists......等等，除了Zend引擎中定义的内部函数，PHP扩展中也提供了大量内部函数，我们也可以灵活的通过扩展自行定制。

#### 3.2.2.1 内部函数结构
上一节介绍`zend_function`为union，其中`internal_function`就是内部函数用到的，具体结构：
```c
//zend_complie.h
typedef struct _zend_internal_function {
    /* Common elements */
    zend_uchar type;
    zend_uchar arg_flags[3]; /* bitset of arg_info.pass_by_reference */
    uint32_t fn_flags;
    zend_string* function_name;
    zend_class_entry *scope;
    zend_function *prototype;
    uint32_t num_args;
    uint32_t required_num_args;
    zend_internal_arg_info *arg_info;
    /* END of common elements */

    void (*handler)(INTERNAL_FUNCTION_PARAMETERS); //函数指针，展开：void (*handler)(zend_execute_data *execute_data, zval *return_value)
    struct _zend_module_entry *module;
    void *reserved[ZEND_MAX_RESERVED_RESOURCES];
} zend_internal_function;
```
`zend_internal_function`头部是一个与`zend_op_array`完全相同的common结构。

下面看下如何定义一个内部函数。

#### 3.2.2.2 定义与注册
内部函数与用户自定义函数冲突，用户无法在PHP代码中覆盖内部函数，执行PHP脚本时会提示error错误。

内部函数的定义非常简单，我们只需要创建一个普通的C函数，然后创建一个`zend_internal_function`结构添加到 __EG(function_table)__ (也可能是CG(function_table),取决于在哪一阶段注册)中即可使用，内部函数 __通常__ 情况下是在php_module_startup阶段注册的，这里之所以说通常是按照标准的扩展定义，除了扩展提供的方式我们可以在任何阶段自由定义内部函数，当然并不建议这样做。下面我们先不讨论扩展标准的定义方式，我们先自己尝试下如何注册一个内部函数。

根据`zend_internal_function`的结构我们知道需要定义一个handler：
```c
void qp_test(INTERNAL_FUNCTION_PARAMETERS)
{
    printf("call internal function 'qp_test'\n");
}
```
然后创建一个内部函数结构(我们在扩展PHP_MINIT_FUNCTION方法中注册，也可以在其他位置)：
```c
PHP_MINIT_FUNCTION(xxxxxx)
{
    zend_string *lowercase_name;
    zend_function *reg_function;

    //函数名转小写，因为php的函数不区分大小写
    lowercase_name = zend_string_alloc(7, 1);
    zend_str_tolower_copy(ZSTR_VAL(lowercase_name), "qp_test", 7);
    lowercase_name = zend_new_interned_string(lowercase_name); 

    reg_function = malloc(sizeof(zend_internal_function));
    reg_function->internal_function.type = ZEND_INTERNAL_FUNCTION; //定义类型为内部函数
    reg_function->internal_function.function_name = lowercase_name;
    reg_function->internal_function.handler = qp_test;

    zend_hash_add_ptr(CG(function_table), lowercase_name, reg_function); //注册到CG(function_table)符号表中
}
```
接着编译、安装扩展，测试：
```php
qp_test();
```
结果输出：
`call internal function 'qp_test'`

这样一个内部函数就定义完成了。这里有一个地方需要注意的我们把这个函数注册到 __CG(function_table)__ 中去了，而不是 __EG(function_table)__ ，这是因为在`php_request_startup`阶段会把 __CG(function_table)__ 赋值给 __EG(function_table)__ 。

上面的过程看着比较简单，但是在实际应用中不要这样做，PHP提供给我们一套标准的定义方式，接下来看下如何在扩展中按照官方方式提供一个内部函数。

首先也是定义C函数，这个通过`PHP_FUNCTION`宏定义：
```c
PHP_FUNCTION(qp_test)
{
    printf("call internal function 'qp_test'\n");
}
```
然后是注册过程，这个只需要我们将所有的函数数组添加到扩展结构`zend_module_entry.functions`即可，扩展加载过程中会自动进行函数注册(见1.2节)，不需要我们干预：
```c
const zend_function_entry xxxx_functions[] = {
        PHP_FE(qp_test,   NULL)
        PHP_FE_END
};

zend_module_entry xxxx_module_entry = {
    STANDARD_MODULE_HEADER,
    "扩展名称",
    xxxx_functions,
    PHP_MINIT(timeout),
    PHP_MSHUTDOWN(timeout),
    PHP_RINIT(timeout),     /* Replace with NULL if there's nothing to do at request start */
    PHP_RSHUTDOWN(timeout), /* Replace with NULL if there's nothing to do at request end */
    PHP_MINFO(timeout),
    PHP_TIMEOUT_VERSION,
    STANDARD_MODULE_PROPERTIES
};
```
关于更多扩展中函数相关的用法会在后面扩展开发一章中详细介绍，这里不再展开。


