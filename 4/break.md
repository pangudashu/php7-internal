## 4.4 中断及跳转
PHP中的中断及跳转语句主要有break、continue、goto，这几种语句的实现基础都是跳转。

### 4.4.1 break与continue
break用于结束当前for、foreach、while、do-while 或者 switch 结构的执行；continue用于跳过本次循环中剩余代码，进行下一轮循环。break、continue是非常相像的，它们都可以接受一个可选数字参数来决定跳过的循环层数，两者的不同点在于break是跳到循环结束的位置，而continue是跳到循环判断条件的位置，本质在于跳转位置的不同。

break、continue的实现稍微有些复杂，

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
