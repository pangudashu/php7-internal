## 4.1 类型转换
PHP是弱类型语言，不需要明确的定义变量的类型，变量的类型根据使用时的上下文所决定，也就是变量会根据具体的使用情况自动切换自己的类型，比如加法操作，PHP会将两个相加的值转为long、double再进行加和，每种类型转为另外一种类型都有固定的规则，当某个操作发现类型不符时就会按照这个规则进行转换，这个规则正是弱类型实现的基础。

除了自动类型转换，PHP还提供了一种强制的转换方式:
* (int)/(integer)：转换为整形 integer
* (bool)/(boolean)：转换为布尔类型 boolean
* (float)/(double)/(real)：转换为浮点型 float
* (string)：转换为字符串 string
* (array)：转换为数组 array
* (object)：转换为对象 object
* (unset)：转换为 NULL

无论是自动类型转换还是强制类型转换，不是每种类型都可以转为任意其他类型，另外，转换时并不是直接在原zval上修改，而是会新分配一个zval，而且很多自动类型转换只是在逻辑处理时按照自己需要的类型去处理，这种连新的zval都不需要。

### 4.1.1 转换为NULL
这种转换比较简单，任意类型都可以转为NULL，转换时直接将新的zval类型设置为`IS_NULL`即可。

### 4.1.2 转换为布尔型
当转换为 boolean 时，根据原值的TRUE、FALSE决定转换后的结果，以下值被认为是 FALSE：
* 布尔值 FALSE 本身
* 整型值 0
* 浮点型值 0.0
* 空字符串，以及字符串 "0"
* 空数组
* NULL

所有其它值都被认为是 TRUE，比如资源、对象(这里指默认情况下，因为可以通过扩展改变这个规则)。

判断一个值是否为true的操作：
```c
static zend_always_inline int i_zend_is_true(zval *op)
{
    int result = 0;

again:
    switch (Z_TYPE_P(op)) {
        case IS_TRUE:
            result = 1;
            break;
        case IS_LONG:
            //非0即真
            if (Z_LVAL_P(op)) {
                result = 1;
            }
            break;
        case IS_DOUBLE:
            if (Z_DVAL_P(op)) {
                result = 1;
            }
            break;
        case IS_STRING:
            //非空字符串及"0"外都为true
            if (Z_STRLEN_P(op) > 1 || (Z_STRLEN_P(op) && Z_STRVAL_P(op)[0] != '0')) {
                result = 1;
            }
            break;
        case IS_ARRAY:
            //非空数组为true
            if (zend_hash_num_elements(Z_ARRVAL_P(op))) {
                result = 1;
            }
            break;
        case IS_OBJECT:
            //默认情况下始终返回true
            result = zend_object_is_true(op);
            break;
        case IS_RESOURCE:
            //合法资源就是true
            if (EXPECTED(Z_RES_HANDLE_P(op))) {
                result = 1;
            }
        case IS_REFERENCE:
            op = Z_REFVAL_P(op);
            goto again;
            break;
        default:
            break;
    }
    return result;
}
```
在扩展中可以通过`convert_to_boolean()`这个函数直接将原zval转为bool型，转换时的判断逻辑与`i_zend_is_true()`一致。

### 4.1.3 转换为整型

### 4.1.4 转换为浮点型

### 4.1.5 转换为字符串

### 4.1.6 转换为数组

### 4.1.7 转换为对象

### 4.1.8 转换为资源
