# Zend引擎执行过程

前面分析了Zend的编译过程以及PHP用户函数的实现，接下来分析下Zend引擎的执行过程。

## 1、数据结构
执行流程中有几个重要的数据结构，先看下这几个结构。

### 1.1、opcode
opcode是将PHP代码编译产生的Zend虚拟机可识别的指令，php7共有173个opcode，定义在`zend_vm_opcodes.h`中，PHP中的所有语法实现都是由这些opcode组成的。

```c
struct _zend_op {
    const void *handler; //对应执行的C语言function，即每条opcode都有一个C function处理
    znode_op op1;   //操作数1
    znode_op op2;   //操作数2
    znode_op result; //返回值
    uint32_t extended_value; 
    uint32_t lineno; 
    zend_uchar opcode;  //opcode指令
    zend_uchar op1_type; //操作数1类型
    zend_uchar op2_type; //操作数2类型
    zend_uchar result_type; //返回值类型
};
```

### 1.2、zend_op_array
`zend_op_array`是Zend引擎执行阶段的输入，是opcode的集合(当然并不仅仅如此)。

![zend_op_array](img/zend_op_array.png)

```c
truct _zend_op_array {
    /* Common elements */
    zend_uchar type; //标识函数类型：1为PHP内部函数(扩展或内核提供的函数)、2为用户自定义函数(即PHP代码中写的function)
    zend_uchar arg_flags[3]; /* bitset of arg_info.pass_by_reference */
    uint32_t fn_flags;
    zend_string *function_name; //函数名
    zend_class_entry *scope; //所属class
    zend_function *prototype;
    uint32_t num_args; //参数数量
    uint32_t required_num_args; //必传参数数量
    zend_arg_info *arg_info; //参数信息
    /* END of common elements */

    uint32_t *refcount;

    uint32_t this_var;

    uint32_t last;
    zend_op *opcodes; //opcode指令数组

    int last_var;
    uint32_t T; //临时变量数
    zend_string **vars; //PHP变量名列表

    int last_brk_cont;
    int last_try_catch;
    zend_brk_cont_element *brk_cont_array;
    zend_try_catch_element *try_catch_array;

    /* static variables support */
    HashTable *static_variables; //静态变量符号表

    zend_string *filename; //PHP文件路径
    uint32_t line_start;
    uint32_t line_end;
    zend_string *doc_comment;
    uint32_t early_binding; /* the linked list of delayed declarations */

    int last_literal; 
    zval *literals; //字面量(常量)数组

    int  cache_size;
    void **run_time_cache;

    void *reserved[ZEND_MAX_RESERVED_RESOURCES];
};

```

### 1.3、zend_executor_globals
`zend_executor_globals executor_globals`是PHP整个生命周期中最主要的一个结构，是一个全局变量，在main执行前分配(非ZTS下)，直到PHP退出，它记录着当前请求全部的信息，经常见到的一个宏`EG`操作的就是这个结构。
```c
//zend_compile.c
#ifndef ZTS
ZEND_API zend_compiler_globals compiler_globals;
ZEND_API zend_executor_globals executor_globals;
#endif

//zend_globals_macros.h
# define EG(v) (executor_globals.v)
```
`zend_executor_globals`结构非常大，定义在`zend_globals.h`中，比较重要的几个字段含义如下图所示：

![EG](img/EG.png)

### 1.4、zend_execute_data
`zend_execute_data`是执行过程中最核心的一个结构，它代表了当前的作用域、代码的执行位置以及局部变量的分配等等，后面分析具体执行流程的时候会详细分析其作用。

```c
#define EX(element)             ((execute_data)->element)

//zend_compile.h
struct _zend_execute_data {
    const zend_op       *opline;  //指向当前执行的opcode，初始时指向zend_op_array起始位置
    zend_execute_data   *call;             /* current call                   */
    zval                *return_value;  //返回值指针 */
    zend_function       *func;          //当前执行的函数（非函数调用时为空）
    zval                 This;          //this + call_info + num_args
    zend_class_entry    *called_scope;  //当前call的类
    zend_execute_data   *prev_execute_data; //函数调用时指向调用位置作用空间
    zend_array          *symbol_table; //全局变量符号表
#if ZEND_EX_USE_RUN_TIME_CACHE
    void               **run_time_cache;   /* cache op_array->run_time_cache */
#endif
#if ZEND_EX_USE_LITERALS
    zval                *literals;  //字面量数组，与func.op_array->literals相同
#endif
};
```

## 2、opcode执行方式
PHP代码编译为opcode数组：zend_op_array，然后逐条执行，在执行的方式上zend提供了三种不同的方式：CALL、SWITCH、GOTO，默认方式为CALL，这个是什么意思呢？

每个opcode都代表了一些特定的处理操作，这个东西怎么提供呢？一种是把每种opcode负责的工作封装成一个function，然后执行器循环执行即可，这就是`CALL`模式的工作方式；另外一种是把所有opcode的处理方式通过C语言里面的label标签区分开，然后执行器执行的时候goto到相应的位置处理，这就是`GOTO`模式的工作方式；最后还有一种方式是把所有的处理方式写到一个switch下，然后通过case不同的opcode执行具体的操作，这就是`SWITCH`模式的工作方式。

假设opcode数组是这个样子：
```c
int op_array[] = {
    opcode_1,
    opcode_2,
    opcode_3,
    ...
};
```
各模式下的工作过程类似这样：
```c
//CALL模式
void opcode_1_handler() {...}

void opcode_2_handler() {...}
...

void execute(int []op_array)
{
    void *opcode_handler_list[] = {&opcode_1_handler, &opcode_2_handler, ...};

    while(1){
        void handler = opcode_handler_list[op_array[i]];
        handler(); //call handler
        i++;
    }
}

//GOTO模式
void execute(int []op_array)
{
    while(1){
        goto opcode_xx_handler_label;
    }

opcode_1_handler_label:
    ...

opcode_2_handler_label:
    ...
...
}

//SWITCH模式
void execute(int []op_array)
{
    while(1){
        switch(op_array[i]){
            case opcode_1:
                ...
            case opcode_2:
                ...
            ...
        }

        i++;
    }
}

```
三种模式效率是不同的，GOTO最快，怎么选择其它模式呢？下载PHP源码后不要直接编译，Zend目录下有个文件：`zend_vm_gen.php`，在编译PHP前执行：`php zend_vm_gen.php --with-vm-kind=CALL|SWITCH|GOTO`，这个脚本将重新生成:`zend_vm_opcodes.h`、`zend_vm_opcodes.c`、`zend_vm_execute.h`三个文件覆盖原来的，然后再编译PHP即可。

后面分析的过程使用的都是默认模式`CALL`。

## 3、执行流程
Zend的executor与linux二进制程序执行的过程是非常类似的，在C程序执行时有两个寄存器ebp、esp分别指向当前作用栈的栈顶、栈底，局部变量全部分配在当前栈，函数调用、返回通过`call`、`ret`指令完成，调用时`call`将当前执行位置压入栈中，返回时`ret`将之前执行位置出栈，跳回旧的位置继续执行，在Zend VM中`zend_execute_data`就扮演了这两个角色，`zend_execute_data.prev_execute_data`保存的是调用方的信息，实现了`call/ret`，`zend_execute_data`后面会分配额外的内存空间用于局部变量的存储，实现了`ebp/esp`的作用。

注意：在执行前分配内存时并不仅仅是分配了`zend_execute_data`大小的空间，除了`sizeof(zend_execute_data)`外还会额外申请一块空间，用于分配局部变量、临时(中间)变量等，具体的分配过程下面会讲到。

__Zend执行opcode的简略过程：__
* __step1:__ 为当前作用域分配一块内存，充当运行栈，zend_execute_data结构、所有局部变量、中间变量等等都在此内存上分配
* __step2:__ 初始化全局变量符号表，然后将全局执行位置指针EG(current_execute_data)指向step1新分配的zend_execute_data，然后将zend_execute_data.opline指向op_array的起始位置
* __step3:__ 从EX(opline)开始调用各opcode的C处理handler(即_zend_op.handler)，每执行完一条opcode将`EX(opline)++`继续执行下一条，直到执行完全部opcode，函数/类成员方法调用、if的执行过程：
    * __step3.1:__ if语句将根据条件的成立与否决定`EX(opline) + offset`所加的偏移量，实现跳转
    * __step3.2:__ 如果是函数调用，则首先从EG(function_table)中根据function_name取出此function对应的编译完成的zend_op_array，然后像step1一样新分配一个zend_execute_data结构，将EG(current_execute_data)赋值给新结构的`prev_execute_data`，再将EG(current_execute_data)指向新的zend_execute_data，最后从新的`zend_execute_data.opline`开始执行，切换到函数内部，函数执行完以后将EG(current_execute_data)重新指向EX(prev_execute_data)，释放分配的运行栈，销毁局部变量，继续从原来函数调用的位置执行
    * __step3.3:__ 类方法的调用与函数基本相同，后面分析对象实现的时候再详细分析
* __step4:__ 全部opcode执行完成后将step1分配的内存释放，这个过程会将所有的局部变量"销毁"，执行阶段结束

![zend_execute](img/zend_execute_data.png)

接下来详细看下整个流程。

Zend执行入口为位于`zend_vm_execute.h`文件中的__zend_execute()__：

```c
ZEND_API void zend_execute(zend_op_array *op_array, zval *return_value)
{
    zend_execute_data *execute_data;

    if (EG(exception) != NULL) {
        return;
    }

    //分配zend_execute_data
    execute_data = zend_vm_stack_push_call_frame(ZEND_CALL_TOP_CODE,
            (zend_function*)op_array, 0, zend_get_called_scope(EG(current_execute_data)), zend_get_this_object(EG(current_execute_data)));
    if (EG(current_execute_data)) {
        execute_data->symbol_table = zend_rebuild_symbol_table();
    } else {
        execute_data->symbol_table = &EG(symbol_table);
    }
    EX(prev_execute_data) = EG(current_execute_data); //=> execute_data->prev_execute_data = EG(current_execute_data);
    i_init_execute_data(execute_data, op_array, return_value); //初始化execute_data
    zend_execute_ex(execute_data); //执行opcode
    zend_vm_stack_free_call_frame(execute_data); //释放execute_data:销毁所有的PHP变量
}

```
上面的过程分为四步：

#### (1)分配stack
由`zend_vm_stack_push_call_frame`函数分配一块用于当前作用域的内存空间，返回结果是`zend_execute_data`的起始位置。
```c
//zend_execute.h
static zend_always_inline zend_execute_data *zend_vm_stack_push_call_frame(uint32_t call_info, zend_function *func, uint32_t num_args, ...)
{
    uint32_t used_stack = zend_vm_calc_used_stack(num_args, func);

    return zend_vm_stack_push_call_frame_ex(used_stack, call_info,
        func, num_args, called_scope, object);
}
```
首先根据`zend_execute_data`、当前`zend_op_array`中局部/临时变量数计算需要的内存空间：
```c
//zend_execute.h
static zend_always_inline uint32_t zend_vm_calc_used_stack(uint32_t num_args, zend_function *func)
{
    uint32_t used_stack = ZEND_CALL_FRAME_SLOT + num_args; //内部函数只用这么多，临时变量是编译过程中根据PHP的代码优化出的值，比如:`"hi~".time()`，而在内部函数中则没有这种情况

    if (EXPECTED(ZEND_USER_CODE(func->type))) { //在php脚本中写的function
        used_stack += func->op_array.last_var + func->op_array.T - MIN(func->op_array.num_args, num_args);
    }
    return used_stack * sizeof(zval);
}

//zend_compile.h
#define ZEND_CALL_FRAME_SLOT \
    ((int)((ZEND_MM_ALIGNED_SIZE(sizeof(zend_execute_data)) + ZEND_MM_ALIGNED_SIZE(sizeof(zval)) - 1) / ZEND_MM_ALIGNED_SIZE(sizeof(zval))))
```
回想下前面编译阶段zend_op_array的结果，在编译过程中已经确定当前作用域下有多少个局部变量(func->op_array.last_var)、临时/中间/无用变量(func->op_array.T)，从而在执行之初就将他们全部分配完成：

* __last_var__：PHP代码中定义的变量数，zend_op.op{1|2}_type = IS_CV 或 result_type & IS_CV的全部数量
* __T__：表示用到的临时变量、无用变量等，zend_op.op{1|2}_type = IS_TMP_VAR|IS_VAR 或resulte_type & (IS_TMP_VAR|IS_VAR)的全部数量

比如赋值操作：`$a = 1234;`，编译后`last_var = 1，T = 1`，`last_var`有`$a`，这里为什么会有`T`？因为赋值语句有一个结果返回值，只是这个值没有用到，假如这么用结果就会用到了`if(($a = 1234) == true){...}`，这时候`$a = 1234;`的返回结果类型是`IS_VAR`，记在`T`上。

`num_args`为函数调用时的实际传入参数数量，`func->op_array.num_args`为全部参数数量，所以`MIN(func->op_array.num_args, num_args)`等于`num_args`，在自定义函数中`used_stack=ZEND_CALL_FRAME_SLOT + func->op_array.last_var + func->op_array.T`，而在调用内部函数时则只需要分配实际传入参数的空间即可，内部函数不会有临时变量的概念。

最终分配的内存空间如下图：

![var_T](img/var_T.png)

这里实际分配内存时并不是直接`malloc`的，还记得上面EG结构中有个`vm_stack`吗？实际内存是从这里获取的，每次从`EG(vm_stack_top)`处开始分配，分配完再将此指针指向`EG(vm_stack_top) + used_stack`，这里不再对vm_stack作更多分析，更下层实际就是Zend的内存池(zend_alloc.c)，后面也会单独分析。

```c
static zend_always_inline zend_execute_data *zend_vm_stack_push_call_frame_ex(uint32_t used_stack, ...)
{
    zend_execute_data *call = (zend_execute_data*)EG(vm_stack_top);
    ...
    
    //当前vm_stack是否够用
    if (UNEXPECTED(used_stack > (size_t)(((char*)EG(vm_stack_end)) - (char*)call))) {
        call = (zend_execute_data*)zend_vm_stack_extend(used_stack); //新开辟一块vm_stack
        ...
    }else{ //空间够用，直接分配
        EG(vm_stack_top) = (zval*)((char*)call + used_stack);
        ...
    }

    call->func = func;
    ...
    return call;
}
```

#### (2)初始化execute_data
这一步的操作主要是设置几个指针:`opline`、`call`、`return_value`，同时将PHP的全局变量添加到`EG(symbol_table)`中去：
```c
//zend_execute.c
static zend_always_inline void i_init_execute_data(zend_execute_data *execute_data, zend_op_array *op_array, zval *return_value)
{
    EX(opline) = op_array->opcodes;
    EX(call) = NULL;
    EX(return_value) = return_value;

    if (UNEXPECTED(EX(symbol_table) != NULL)) {
        ...
        zend_attach_symbol_table(execute_data);//将全局变量添加到EG(symbol_table)中一份
    }else{ //这个分支的情况还未深入分析，后面碰到再补充
        ...
    }
}
```

#### (3)执行opcode
这一步开始具体执行opcode指令，这里调用的是`zend_execute_ex`，这是一个函数指针，如果此指针没有被任何扩展重新定义那么将由默认的`execute_ex`处理：
```c
# define ZEND_OPCODE_HANDLER_ARGS_PASSTHRU execute_data

ZEND_API void execute_ex(zend_execute_data *ex)
{
    zend_execute_data *execute_data = ex;

    while(1) {
        int ret;
        if (UNEXPECTED((ret = ((opcode_handler_t)EX(opline)->handler)(execute_data /*ZEND_OPCODE_HANDLER_ARGS_PASSTHRU*/)) != 0)) {
            if (EXPECTED(ret > 0)) { //调到新的位置执行：函数调用时的情况
                execute_data = EG(current_execute_data);
            }else{
                return;
            }
        }
    }
}
```
大概的执行过程上面已经介绍过了，这里只分析下整体执行流程，至于PHP各语法具体的handler处理后面会单独列一章详细分析。

这里还有一个问题是：handler是如何根据opcode索引到的？opcode的数值各不相同，同时可以根据两个zend_op的类型设置不同的处理handler，因此每个opcode指令最多有20个对应的处理handler，所有的handler按照opcode数值的顺序定义在一个大数组中:`zend_opcode_handlers`，每25个为同一个opcode，如果对应的op_type类型handler则可以设置为空：
```c
void zend_init_opcodes_handlers(void)
{
    static const void *labels[] = {
        ZEND_NOP_SPEC_HANDLER,
        ZEND_NOP_SPEC_HANDLER,
        ...
    }；
    zend_opcode_handlers = labels;
}
```
索引的算法：
```c
static const void *zend_vm_get_opcode_handler(zend_uchar opcode, const zend_op* op)
{
        static const int zend_vm_decode[] = {
            _UNUSED_CODE, /* 0              */
            _CONST_CODE,  /* 1 = IS_CONST   */
            _TMP_CODE,    /* 2 = IS_TMP_VAR */
            _UNUSED_CODE, /* 3              */
            _VAR_CODE,    /* 4 = IS_VAR     */
            _UNUSED_CODE, /* 5              */
            _UNUSED_CODE, /* 6              */
            _UNUSED_CODE, /* 7              */
            _UNUSED_CODE, /* 8 = IS_UNUSED  */
            _UNUSED_CODE, /* 9              */
            _UNUSED_CODE, /* 10             */
            _UNUSED_CODE, /* 11             */
            _UNUSED_CODE, /* 12             */
            _UNUSED_CODE, /* 13             */
            _UNUSED_CODE, /* 14             */
            _UNUSED_CODE, /* 15             */
            _CV_CODE      /* 16 = IS_CV     */
        };
        return zend_opcode_handlers[opcode * 25 + zend_vm_decode[op->op1_type] * 5 + zend_vm_decode[op->op2_type]];
}

ZEND_API void zend_vm_set_opcode_handler(zend_op* op)
{
    op->handler = zend_vm_get_opcode_handler(zend_user_opcodes[op->opcode], op);
}

#define _CONST_CODE  0
#define _TMP_CODE    1
#define _VAR_CODE    2
#define _UNUSED_CODE 3
#define _CV_CODE     4
```

#### (4)释放stack
这一步就比较简单了，只是将申请的`zend_execute_data`内存释放给内存池，具体的操作只需要修改几个指针即可：

```c
static zend_always_inline void zend_vm_stack_free_call_frame_ex(uint32_t call_info, zend_execute_data *call)
{
    ZEND_ASSERT_VM_STACK_GLOBAL;

    if (UNEXPECTED(call_info & ZEND_CALL_ALLOCATED)) {
        zend_vm_stack p = EG(vm_stack);

        zend_vm_stack prev = p->prev;

        EG(vm_stack_top) = prev->top;
        EG(vm_stack_end) = prev->end;
        EG(vm_stack) = prev;
        efree(p);

    } else {
        EG(vm_stack_top) = (zval*)call;
    }

    ZEND_ASSERT_VM_STACK_GLOBAL;
}

static zend_always_inline void zend_vm_stack_free_call_frame(zend_execute_data *call)
{
    zend_vm_stack_free_call_frame_ex(ZEND_CALL_INFO(call), call);
}
```


