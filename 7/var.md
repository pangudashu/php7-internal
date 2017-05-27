## 7.7 zval的操作
扩展中经常会用到各种类型的zval，PHP提供了很多宏用于不同类型zval的操作，尽管我们也可以自己操作zval，但这并不是一个好习惯，因为zval有很多其它用途的标识，如果自己去管理这些值将是非常繁琐的一件事，所以我们应该使用PHP提供的这些宏来操作用到的zval。

本节提到的zval泛指zval及各种zend_value。

### 7.7.1 生成不同类型的zval
PHP7将变量的引用计数转移到了具体的value上，所以zval更多的是作为统一的传输格式，很多情况下只是临时性使用，比如函数调用时的传参，最终需要的数据是zval携带的zend_value，函数从zval取得zend_value后就不再关心zval了，这种就可以直接在栈上分配zval。分配完zval后需要将其设置为我们需要的类型以及设置其zend_value，PHP中定义的`ZVAL_XXX()`系列宏就是用来干这个的，这些宏第一个参数z均为要设置的zval的指针，后面为要设置的zend_value。

* __ZVAL_UNDEF(z):__ 表示zval被销毁
* __ZVAL_NULL(z):__ 设置为NULL
* __ZVAL_FALSE(z):__ 设置为false
* __ZVAL_TRUE(z):__ 设置为true
* __ZVAL_BOOL(z, b):__ 设置为布尔型，b为IS_TRUE、IS_FALSE，与上面两个等价
* __ZVAL_LONG(z, l):__ 设置为整形，l类型为zend_long，如：`zval z; ZVAL_LONG(&z, 88);`
* __ZVAL_DOUBLE(z, d):__ 设置为浮点型，d类型为double
* __ZVAL_STR(z, s):__ 设置字符串，将z的value设置为s，s类型为zend_string*，不会增加s的refcount，支持interned strings
* __ZVAL_NEW_STR(z, s):__ 同ZVAL_STR(z, s)，s为普通字符串，不支持interned strings
* __ZVAL_STR_COPY(z, s):__ 将s拷贝到z的value，s类型为zend_string*，同ZVAL_STR(z, s)，这里会增加s的refcount
* __ZVAL_ARR(z, a):__ 设置为数组，a类型为zend_array*
* __ZVAL_NEW_ARR(z):__ 新分配一个数组，主动分配一个zend_array
* __ZVAL_NEW_PERSISTENT_ARR(z):__ 创建持久化数组，通过malloc分配，需要手动释放
* __ZVAL_OBJ(z, o):__ 设置为对象，o类型为zend_object*
* __ZVAL_RES(z, r):__ 设置为资源，r类型为zend_resource*
* __ZVAL_NEW_RES(z, h, p, t):__ 新创建一个资源，h为资源handle，t为type，p为资源ptr指向结构
* __ZVAL_REF(z, r):__ 设置为引用，r类型为zend_reference*
* __ZVAL_NEW_EMPTY_REF(z):__ 新创建一个空引用，没有设置具体引用的value
* __ZVAL_NEW_REF(z, r):__ 新创建一个引用，r为引用的值，类型为zval*
* ...

### 7.7.2 获取zval的值

### 7.7.3 引用计数

