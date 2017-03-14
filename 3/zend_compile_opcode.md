### 3.1.2 抽象语法树编译流程

上一小节我们简单介绍了从PHP代码解析为抽象语法树的过程，这一节我们再介绍下从__抽象语法树->Opcodes__的过程。

语法解析过程的产物保存于CG(AST)，接着zend引擎会把AST进一步编译为__zend_op_array__，它是编译阶段最终的产物，也是执行阶段的输入，后面我们介绍的东西基本都是围绕zend_op_array展开的，我们接着看下语法解析(zendparse())之后的流程:

```c
ZEND_API zend_op_array *compile_file(zend_file_handle *file_handle, int type)
{
    zend_op_array *op_array = NULL; //编译出的opcodes
    ...

    if (open_file_for_scanning(file_handle)==FAILURE) {//文件打开失败
        ...
    } else {
        zend_bool original_in_compilation = CG(in_compilation);
        CG(in_compilation) = 1;

        CG(ast) = NULL;
        CG(ast_arena) = zend_arena_create(1024 * 32);
        if (!zendparse()) { //语法解析
            zval retval_zv;
            zend_file_context original_file_context; //保存原来的zend_file_context
            zend_oparray_context original_oparray_context; //保存原来的zend_oparray_context
            zend_op_array *original_active_op_array = CG(active_op_array);
            op_array = emalloc(sizeof(zend_op_array)); //分配zend_op_array结构
            init_op_array(op_array, ZEND_USER_FUNCTION, INITIAL_OP_ARRAY_SIZE);//初始化op_array
            CG(active_op_array) = op_array; //将当前正在编译op_array指向当前
            ZVAL_LONG(&retval_zv, 1);

            if (zend_ast_process) {
                zend_ast_process(CG(ast));
            }

            zend_file_context_begin(&original_file_context); //初始化CG(file_context)
            zend_oparray_context_begin(&original_oparray_context); //初始化CG(context)
            zend_compile_top_stmt(CG(ast)); //AST->zend_op_array编译流程
            zend_emit_final_return(&retval_zv); //设置最后的返回值
            op_array->line_start = 1;
            op_array->line_end = CG(zend_lineno);
            pass_two(op_array);
            zend_oparray_context_end(&original_oparray_context);
            zend_file_context_end(&original_file_context);

            CG(active_op_array) = original_active_op_array;
        }
        ...
    }
    ...

    return op_array;
}
```
compile_file()操作中有几个保存原来值的操作，这是因为这个函数在PHP脚本执行中并不会只执行一次，主脚本执行时会第一次调用，而include、require也会调用，所以需要先保存当前值，然后执行完再还原回去。

AST->zend_op_array编译是在__zend_compile_top_stmt()__中完成，这个函数是总入口，会被多次递归调用：
```c
//zend_compile.c
void zend_compile_top_stmt(zend_ast *ast)
{
    if (!ast) {
        return;
    }

    if (ast->kind == ZEND_AST_STMT_LIST) { //第一次进来一定是这种类型
        zend_ast_list *list = zend_ast_get_list(ast);
        uint32_t i;
        for (i = 0; i < list->children; ++i) {
            zend_compile_top_stmt(list->child[i]);//list各child语句相互独立，递归编译
        }
        return;
    }

    //各语句编译入口
    zend_compile_stmt(ast);

    if (ast->kind != ZEND_AST_NAMESPACE && ast->kind != ZEND_AST_HALT_COMPILER) {
        zend_verify_namespace();
    }
    //function、class两种情况的处理，非常关键的一步操作，后面分析函数、类实现的章节再详细分析
    if (ast->kind == ZEND_AST_FUNC_DECL || ast->kind == ZEND_AST_CLASS) {
        CG(zend_lineno) = ((zend_ast_decl *) ast)->end_lineno;
        zend_do_early_binding();
    }
}
```
首先从AST的根节点开始编译，根节点类型为ZEND_AST_STMT_LIST，这个类型表示当前节点下有多个独立的节点，各child都是独立的语句生成的节点，所以依次编译即可，直到到达有效节点位置(非ZEND_AST_STMT_LIST节点)，然后调用`zend_compile_stmt`编译当前节点：
```c

```
主要根据不同的节点类型(kind)作不同的处理，我们不会把每种类型的处理都讲一遍，这里还是根据上一节最后的例子挑几个看下具体的处理过程。
```php
$a = 123;
$b = "hi~";

echo $a,$b;
```
zendparse()阶段生成的AST：

![zend_ast](../img/zend_ast.png)


