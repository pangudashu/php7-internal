## 4.4 中断及跳转
PHP中的中断及跳转语句主要有break、continue、goto，这几种语句的实现基础都是跳转。

### 4.4.1 break与continue
break用于结束当前for、foreach、while、do-while 或者 switch 结构的执行；continue用于跳过本次循环中剩余代码，进行下一轮循环。break、continue是非常相像的，它们都可以接受一个可选数字参数来决定跳过的循环层数，两者的不同点在于break是跳到循环结束的位置，而continue是跳到循环判断条件的位置，本质在于跳转位置的不同。

break、continue的实现稍微有些复杂，下面具体介绍下其编译过程。

上一节我们已经介绍过循环语句的编译，其中在各种循环编译过程中有两个特殊操作：zend_begin_loop()、zend_end_loop()，分别在循环编译前以及编译后调用，这两步操作就是为break、continue服务的。

在每层循环编译时都会创建一个`zend_brk_cont_element`的结构：
```c
typedef struct _zend_brk_cont_element {
    int start;
    int cont;
    int brk;
    int parent;
} zend_brk_cont_element;
```
cont记录的是当前循环判断条件opcode起始位置，brk记录的是当前循环结束的位置，parent记录的是父层循环`zend_brk_cont_element`结构的存储位置，也就是说多层嵌套循环会生成一个`zend_brk_cont_element`的链表，每层循环编译结束时更新自己的`zend_brk_cont_element`结构，所以break、continue的处理过程实际就是根据跳出的层级索引到那一层的`zend_brk_cont_element`结构，然后得到它的cont、brk进行相应的opcode跳转。

各循环的`zend_brk_cont_element`结构保存在`zend_op_array->brk_cont_array`数组中，实际这个数组在编译前就已经分配好了，编译各循环时依次申请一个`zend_brk_cont_element`，`zend_op_array->last_brk_cont`记录此数组第一个可用位置，每申请一个元素last_brk_cont就相应的增加1，parent记录的就是父层循环在`zend_op_array->brk_cont_array`中的位置。

示例：
```php
$i = 0;
while(1){
    while(1){
        if($i > 10){
            break 2;
        }
        ++$i
    }
}
```
循环编译完以后对应的内存结构：

![](../img/loop_op.png)

介绍完编译循环结构时为break、continue做的准备，接下来我们具体分析下break、continue的编译。

有了前面的准备，break、continue的编译过程就比较简单了，主要就是各生成一条临时opcode：ZEND_BRK、ZEND_CONT，这条opcode记录着两个重要信息：
* __op1:__ 记录着当前循环`zend_brk_cont_element`结构的存储位置(在循环编译过程中CG(context).current_brk_cont记录着当前循环zend_brk_cont_element的位置)
* __op2:__ 记录着要跳出循环的层级，如果break/continue没有加数字，则默认为1

```c
void zend_compile_break_continue(zend_ast *ast)
{
    zend_ast *depth_ast = ast->child[0];

    zend_op *opline;
    int depth;

    if (depth_ast) {
        zval *depth_zv;
        if (depth_ast->kind != ZEND_AST_ZVAL) {
            zend_error_noreturn(E_COMPILE_ERROR, "'%s' operator with non-constant operand "
                "is no longer supported", ast->kind == ZEND_AST_BREAK ? "break" : "continue");
        }

        depth_zv = zend_ast_get_zval(depth_ast);
        if (Z_TYPE_P(depth_zv) != IS_LONG || Z_LVAL_P(depth_zv) < 1) {
            zend_error_noreturn(E_COMPILE_ERROR, "'%s' operator accepts only positive numbers",
                ast->kind == ZEND_AST_BREAK ? "break" : "continue");
        }

        depth = Z_LVAL_P(depth_zv);
    } else {
        depth = 1;
    }

    if (CG(context).current_brk_cont == -1) {
        zend_error_noreturn(E_COMPILE_ERROR, "'%s' not in the 'loop' or 'switch' context",
            ast->kind == ZEND_AST_BREAK ? "break" : "continue");
    } else {
        if (!zend_handle_loops_and_finally_ex(depth)) {
            zend_error_noreturn(E_COMPILE_ERROR, "Cannot '%s' %d level%s",
                ast->kind == ZEND_AST_BREAK ? "break" : "continue",
                depth, depth == 1 ? "" : "s");
        }
    }
    opline = zend_emit_op(NULL, ast->kind == ZEND_AST_BREAK ? ZEND_BRK : ZEND_CONT, NULL, NULL);
    opline->op1.num = CG(context).current_brk_cont; //所在循环层
    opline->op2.num = depth;  //要跳出的层数
}
```

### 4.4.2 goto
