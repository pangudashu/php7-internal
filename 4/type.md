## 4.1 类型转换
PHP是弱类型语言，不需要明确的定义变量的类型，变量的类型根据使用时的上下文所决定，也就是变量会根据不同表达式所需要的类型自动转换，比如求和，PHP会将两个相加的值转为long、double再进行加和。每种类型转为另外一种类型都有固定的规则，当某个操作发现类型不符时就会按照这个规则进行转换，这个规则正是弱类型实现的基础。

除了自动类型转换，PHP还提供了一种强制的转换方式:
* (int)/(integer)：转换为整形 integer
* (bool)/(boolean)：转换为布尔类型 boolean
* (float)/(double)/(real)：转换为浮点型 float
* (string)：转换为字符串 string
* (array)：转换为数组 array
* (object)：转换为对象 object
* (unset)：转换为 NULL

无论是自动类型转换还是强制类型转换，不是每种类型都可以转为任意其他类型。

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
其它类型转为整形的转换规则：
* NULL：转为0
* 布尔型：false转为0，true转为1
* 浮点型：向下取整，比如：`(int)2.8 => 2`
* 字符串：就是C语言strtoll()的规则，如果字符串以合法的数值开始，则使用该数值，否则其值为 0（零），合法数值由可选的正负号，后面跟着一个或多个数字（可能有小数点），再跟着可选的指数部分
* 数组：很多操作不支持将一个数组自动整形处理，比如：`array() + 2`，将报error错误，但可以强制把数组转为整形，非空数组转为1，空数组转为0，没有其他值
* 对象：与数组类似，很多操作也不支持将对象自动转为整形，但有些操作只会抛一个warning警告，还是会把对象转为1操作的，这个需要看不同操作的处理情况
* 资源：转为分配给这个资源的唯一编号

具体处理：
```c
ZEND_API zend_long ZEND_FASTCALL _zval_get_long_func(zval *op)
{
try_again: 
    switch (Z_TYPE_P(op)) {
        case IS_NULL:
        case IS_FALSE:
            return 0;
        case IS_TRUE:
            return 1;
        case IS_RESOURCE:
            //资源将转为zend_resource->handler
            return Z_RES_HANDLE_P(op);
        case IS_LONG:
            return Z_LVAL_P(op);
        case IS_DOUBLE:
            return zend_dval_to_lval(Z_DVAL_P(op));
        case IS_STRING:
            //字符串的转换调用C语言的strtoll()处理
            return ZEND_STRTOL(Z_STRVAL_P(op), NULL, 10);
        case IS_ARRAY:
            //根据数组是否为空转为0,1
            return zend_hash_num_elements(Z_ARRVAL_P(op)) ? 1 : 0;
        case IS_OBJECT:
            {   
                zval dst;
                convert_object_to_type(op, &dst, IS_LONG, convert_to_long);
                if (Z_TYPE(dst) == IS_LONG) {
                    return Z_LVAL(dst);
                } else {
                    //默认情况就是1
                    return 1;
                }
            }
        case IS_REFERENCE:
            op = Z_REFVAL_P(op);
            goto try_again;
            EMPTY_SWITCH_DEFAULT_CASE()
    }
    return 0;
}
```
### 4.1.4 转换为浮点型
除字符串类型外，其它类型转换规则与整形基本一致，就是整形转换结果加了一位小数，字符串转为浮点数由`zend_strtod()`完成，这个函数非常长，定义在`zend_strtod.c`中，这里不作说明。

### 4.1.5 转换为字符串
一个值可以通过在其前面加上 (string) 或用 strval() 函数来转变成字符串。在一个需要字符串的表达式中，会自动转换为 string，比如在使用函数 echo 或 print 时，或在一个变量和一个 string 进行比较时，就会发生这种转换。

```c
ZEND_API zend_string* ZEND_FASTCALL _zval_get_string_func(zval *op)
{
try_again:
    switch (Z_TYPE_P(op)) {
        case IS_UNDEF:
        case IS_NULL:
        case IS_FALSE:
            //转为空字符串""
            return ZSTR_EMPTY_ALLOC();
        case IS_TRUE:
            //转为"1"
            ...
            return zend_string_init("1", 1, 0);
        case IS_RESOURCE: {
            //转为"Resource id #xxx"
            ...
            len = snprintf(buf, sizeof(buf), "Resource id #" ZEND_LONG_FMT, (zend_long)Z_RES_HANDLE_P(op));
            return zend_string_init(buf, len, 0);
        }
        case IS_LONG: {
            return zend_long_to_str(Z_LVAL_P(op));
        }
        case IS_DOUBLE: {
            return zend_strpprintf(0, "%.*G", (int) EG(precision), Z_DVAL_P(op));
        }
        case IS_ARRAY:
            //转为"Array"，但是报Notice
            zend_error(E_NOTICE, "Array to string conversion");
            return zend_string_init("Array", sizeof("Array")-1, 0);
        case IS_OBJECT: {
            //报Error错误
            zval tmp;
            ...
            zend_error(EG(exception) ? E_ERROR : E_RECOVERABLE_ERROR, "Object of class %s could not be converted to string", ZSTR_VAL(Z_OBJCE_P(op)->name));
            return ZSTR_EMPTY_ALLOC();
        }
        case IS_REFERENCE:
            op = Z_REFVAL_P(op);
            goto try_again;
        case IS_STRING:
            return zend_string_copy(Z_STR_P(op));
        EMPTY_SWITCH_DEFAULT_CASE()
    }
    return NULL;
}
```

### 4.1.6 转换为数组
如果将一个null、integer、float、string、boolean 和 resource 类型的值转换为数组，将得到一个仅有一个元素的数组，其下标为 0，该元素即为此标量的值。换句话说，(array)$scalarValue 与 array($scalarValue) 完全一样。

如果一个 object 类型转换为 array，则结果为一个数组，数组元素为该对象的全部属性，包括public、private、protected，其中private的属性转换后的key加上了类名作为前缀，protected属性的key加上了"*"作为前缀，举例来看：
```c
class test {
	private $a = 123;
	public $b = "bbb";
	protected $c = "ccc";
}
$obj = new test;
print_r((array)$obj);
======================
Array
(
    [testa] => 123
    [b] => bbb
    [*c] => ccc
)
```
转换时的处理：
```c
ZEND_API void ZEND_FASTCALL convert_to_array(zval *op)
{
	try_again:
    switch (Z_TYPE_P(op)) {
        case IS_ARRAY:
            break;
        case IS_OBJECT:
			...
			if (Z_OBJ_HT_P(op)->get_properties) {
				//获取所有属性数组
				HashTable *obj_ht = Z_OBJ_HT_P(op)->get_properties(op);
				//将数组内容拷贝到新数组
				...
			}
		case IS_NULL:
            ZVAL_NEW_ARR(op);
			//转为空数组
            zend_hash_init(Z_ARRVAL_P(op), 8, NULL, ZVAL_PTR_DTOR, 0);
            break;
        case IS_REFERENCE:
            zend_unwrap_reference(op);
            goto try_again;
        default:
            convert_scalar_to_array(op);
            break;
    }
}

//其他标量类型转array
static void convert_scalar_to_array(zval *op)
{
    zval entry;

    ZVAL_COPY_VALUE(&entry, op);
	//新分配一个数组，将原值插入数组
    ZVAL_NEW_ARR(op);
    zend_hash_init(Z_ARRVAL_P(op), 8, NULL, ZVAL_PTR_DTOR, 0);
    zend_hash_index_add_new(Z_ARRVAL_P(op), 0, &entry);
}
```

### 4.1.7 转换为对象

### 4.1.8 转换为资源
