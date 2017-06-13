## 7.7 zval的操作
扩展中经常会用到各种类型的zval，PHP提供了很多宏用于不同类型zval的操作，尽管我们也可以自己操作zval，但这并不是一个好习惯，因为zval有很多其它用途的标识，如果自己去管理这些值将是非常繁琐的一件事，所以我们应该使用PHP提供的这些宏来操作用到的zval。

### 7.7.1 新生成各类型zval
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

### 7.7.2 获取zval的值及类型
zval的类型通过`Z_TYPE(zval)`、`Z_TYPE_P(zval*)`两个宏获取，这个值取的就是`zval.u1.v.type`，但是设置时不要只修改这个type，而是要设置typeinfo，因为zval还有其它的标识需要设置，比如是否使用引用计数、是否可被垃圾回收、是否可被复制等等。

内核提供了`Z_XXX(zval)`、`Z_XXX_P(zval*)`系列的宏用于获取不同类型zval的value。

* __Z_LVAL(zval)、Z_LVAL_P(zval_p):__ 返回zend_long
* __Z_DVAL(zval)、Z_DVAL_P(zval_p):__ 返回double
* __Z_STR(zval)、Z_STR_P(zval_p):__ 返回zend_string*
* __Z_STRVAL(zval)、Z_STRVAL_P(zval_p):__ 返回char*，即：zend_string->val
* __Z_STRLEN(zval)、Z_STRLEN_P(zval_p):__ 获取字符串长度
* __Z_STRHASH(zval)、Z_STRHASH_P(zval_p):__ 获取字符串的哈希值
* __Z_ARR(zval)、Z_ARR_P(zval_p)、Z_ARRVAL(zval)、Z_ARRVAL_P(zval_p):__ 返回zend_array*
* __Z_OBJ(zval)、Z_OBJ_P(zval_p):__ 返回zend_object*
* __Z_OBJ_HT(zval)、Z_OBJ_HT_P(zval_p):__ 返回对象的zend_object_handlers，即zend_object->handlers
* __Z_OBJ_HANDLER(zval, hf)、Z_OBJ_HANDLER_P(zv_p, hf):__ 获取对象各操作的handler指针，hf为write_property、read_property等，注意：这个宏取到的为只读，不要试图修改这个值(如：Z_OBJ_HANDLER(obj, write_property) = xxx;)，因为对象的handlers成员前加了const修饰符
* __Z_OBJCE(zval)、Z_OBJCE_P(zval_p):__ 返回对象的zend_class_entry*
* __Z_OBJPROP(zval)、Z_OBJPROP_P(zval_p):__ 获取对象的成员数组
* __Z_RES(zval)、Z_RES_P(zval_p):__ 返回zend_resource*
* __Z_RES_HANDLE(zval)、Z_RES_HANDLE_P(zval_p):__ 返回资源handle
* __Z_RES_TYPE(zval)、Z_RES_TYPE_P(zval_p):__ 返回资源type
* __Z_RES_VAL(zval)、Z_RES_VAL_P(zval_p):__ 返回资源ptr
* __Z_REF(zval)、Z_REF_P(zval_p):__ 返回zend_reference*
* __Z_REFVAL(zval)、Z_REFVAL_P(zval_p):__ 返回引用的zval*

除了这些与PHP变量类型相关的宏之外，还有一些内核自己使用类型的宏：
```c
//获取indirect的zval，指向另一个zval
#define Z_INDIRECT(zval)            (zval).value.zv
#define Z_INDIRECT_P(zval_p)        Z_INDIRECT(*(zval_p))

#define Z_CE(zval)                  (zval).value.ce
#define Z_CE_P(zval_p)              Z_CE(*(zval_p))

#define Z_FUNC(zval)                (zval).value.func
#define Z_FUNC_P(zval_p)            Z_FUNC(*(zval_p))

#define Z_PTR(zval)                 (zval).value.ptr
#define Z_PTR_P(zval_p)             Z_PTR(*(zval_p))
```
### 7.7.3 类型转换
```c
//将原类型转为特定类型，会更改原来的值
ZEND_API void ZEND_FASTCALL convert_to_long(zval *op);
ZEND_API void ZEND_FASTCALL convert_to_double(zval *op);
ZEND_API void ZEND_FASTCALL convert_to_long_base(zval *op, int base);
ZEND_API void ZEND_FASTCALL convert_to_null(zval *op);
ZEND_API void ZEND_FASTCALL convert_to_boolean(zval *op);
ZEND_API void ZEND_FASTCALL convert_to_array(zval *op);
ZEND_API void ZEND_FASTCALL convert_to_object(zval *op);

#define convert_to_cstring(op) if (Z_TYPE_P(op) != IS_STRING) { _convert_to_cstring((op) ZEND_FILE_LINE_CC); }
#define convert_to_string(op) if (Z_TYPE_P(op) != IS_STRING) { _convert_to_string((op) ZEND_FILE_LINE_CC); }

//获取格式化为long的值，不会更改原来的值，op类型为zval*，返回值为zend_long
#define zval_get_long(op) _zval_get_long((op))
//获取格式化为double的值，返回值double
#define zval_get_double(op) _zval_get_double((op))
//获取格式化为string的值，返回值zend_string *
#define zval_get_string(op) _zval_get_string((op))

//字符串转整形
ZEND_API int ZEND_FASTCALL zend_atoi(const char *str, int str_len);
ZEND_API zend_long ZEND_FASTCALL zend_atol(const char *str, int str_len);

//判断是否为true
#define zval_is_true(op) \
    zend_is_true(op)
```
### 7.7.4 引用计数
在扩展中操作与PHP用户空间相关的变量时需要考虑是否需要对其引用计数进行加减，比如下面这个例子：
```php
function test($arr){
    return $arr;
}

$a = array(1,2);
$b = test($a);
```
如果把函数test()用内部函数实现，这个函数接受了一个PHP用户空间传入的数组参数，然后又返回并赋值给了PHP用户空间的另外一个变量，这个时候就需要增加传入数组的refcount，因为这个数组由PHP用户空间分配，函数调用前refcount=1，传到内部函数时相当于赋值给了函数的参数，因此refcount增加了1变为2，这次增加在函数执行完释放参数时会减掉，等返回并赋值给$b后此时共有两个变量指向这个数组，所以内部函数需要增加refcount，增加的引用是给返回值的。test()翻译成内部函数：
```c
PHP_FUNCTION(test)
{   
    zval    *arr;

    if(zend_parse_parameters(ZEND_NUM_ARGS(), "a", &arr) == FAILURE){
        RETURN_FALSE;
    }
    //如果注释掉下面这句将导致core dumped
    Z_TRY_ADDREF_P(arr);
    RETURN_ARR(Z_ARR_P(arr));
} 
```
那么在哪些情况下需要考虑设置引用计数呢？一个关键条件是：操作的是与PHP用户空间相关的变量，包括对用户空间变量的修改、赋值，要明确的一点是引用计数是用来解决多个变量指向同一个value问题的，所以在PHP中来回传递zval的时候就需要考虑下是不是要修改引用计数，下面总结下PHP中常见的会对引用计数进行操作的情况：
* __(1)变量赋值:__ 变量赋值是最常见的情况，一个用到引用计数的变量类型在初始赋值时其refcount=1，如果后面把此变量又赋值给了其他变量那么就会相应的增加其引用计数
* __(2)数组操作：__ 如果把一个变量插入数组中那么就需要增加这个变量的引用计数，如果要删除一个数组元素则要相应的减少其引用
* __(3)函数调用：__ 传参实际可以当做普通的变量赋值，将调用空间的变量赋值给被调函数空间的变量，函数返回时会销毁函数空间的变量，这时又会减掉传参的引用，这两个过程由内核完成，不需要扩展自己处理
* __(4)成员属性：__ 当把一个变量赋值给对象的成员属性时需要增加引用计数

PHP中定义了以下宏用于引用计数的操作：
```c
//获取引用数：pz类型为zval*
#define Z_REFCOUNT_P(pz)            zval_refcount_p(pz)
//设置引用数
#define Z_SET_REFCOUNT_P(pz, rc)    zval_set_refcount_p(pz, rc)
//增加引用
#define Z_ADDREF_P(pz)              zval_addref_p(pz)
//减少引用
#define Z_DELREF_P(pz)              zval_delref_p(pz)

#define Z_REFCOUNT(z)               Z_REFCOUNT_P(&(z))
#define Z_SET_REFCOUNT(z, rc)       Z_SET_REFCOUNT_P(&(z), rc)
#define Z_ADDREF(z)                 Z_ADDREF_P(&(z))
#define Z_DELREF(z)                 Z_DELREF_P(&(z))

//只对使用了引用计数的变量类型增加引用，建议使用这个
#define Z_TRY_ADDREF_P(pz) do {     \
    if (Z_REFCOUNTED_P((pz))) {     \
        Z_ADDREF_P((pz));           \
    }                               \
} while (0)

#define Z_TRY_DELREF_P(pz) do {     \
    if (Z_REFCOUNTED_P((pz))) {     \
        Z_DELREF_P((pz));           \
    }                               \
} while (0)

#define Z_TRY_ADDREF(z)             Z_TRY_ADDREF_P(&(z))
#define Z_TRY_DELREF(z)             Z_TRY_DELREF_P(&(z))
```
这些宏操作类型都是zval或zval*，如果需要操作具体value的引用计数可以使用以下宏：
```c
//直接获取zend_value的引用，可以直接通过这个宏修改value的refcount
#define GC_REFCOUNT(p)              (p)->gc.refcount
```
另外还有几个常用的宏：
```c
//判断zval是否用到引用计数机制
#define Z_REFCOUNTED(zval)           ((Z_TYPE_FLAGS(zval) & IS_TYPE_REFCOUNTED) != 0)
#define Z_REFCOUNTED_P(zval_p)       Z_REFCOUNTED(*(zval_p))

//根据zval获取value的zend_refcounted头部
#define Z_COUNTED(zval)              (zval).value.counted
#define Z_COUNTED_P(zval_p)          Z_COUNTED(*(zval_p))
```
### 7.7.5 字符串操作
PHP中字符串(即：zend_string)操作相关的宏及函数：
```c
//创建zend_string
zend_string *zend_string_init(const char *str, size_t len, int persistent);

//字符串复制，只增加引用
zend_string *zend_string_copy(zend_string *s);

//字符串拷贝，硬拷贝
zend_string *zend_string_dup(zend_string *s, int persistent);

//将字符串按len大小重新分配，会减少s的refcount，返回新的字符串
zend_string *zend_string_realloc(zend_string *s, size_t len, int persistent);

//延长字符串，与zend_string_realloc()类似，不同的是len不能小于s的长度
zend_string *zend_string_extend(zend_string *s, size_t len, int persistent);

//截断字符串，与zend_string_realloc()类似，不同的是len不能大于s的长度
zend_string *zend_string_truncate(zend_string *s, size_t len, int persistent);

//获取字符串refcount
uint32_t zend_string_refcount(const zend_string *s);

//增加字符串refcount
uint32_t zend_string_addref(zend_string *s);

//减少字符串refcount
uint32_t zend_string_delref(zend_string *s);

//释放字符串，减少refcount，为0时销毁
void zend_string_release(zend_string *s);

//销毁字符串，不管引用计数是否为0
void zend_string_free(zend_string *s);

//比较两个字符串是否相等，区分大小写，memcmp()
zend_bool zend_string_equals(zend_string *s1, zend_string *s2);

//比较两个字符串是否相等，不区分大小写
#define zend_string_equals_ci(s1, s2) \
    (ZSTR_LEN(s1) == ZSTR_LEN(s2) && !zend_binary_strcasecmp(ZSTR_VAL(s1), ZSTR_LEN(s1), ZSTR_VAL(s2), ZSTR_LEN(s2)))

//其它宏，zstr类型为zend_string*
#define ZSTR_VAL(zstr)  (zstr)->val //获取字符串
#define ZSTR_LEN(zstr)  (zstr)->len //获取字符串长度
#define ZSTR_H(zstr)    (zstr)->h   //获取字符串哈希值
#define ZSTR_HASH(zstr) zend_string_hash_val(zstr) //计算字符串哈希值
```
除了上面这些，还有很多字符串大小转换、字符串比较的API定义在zend_operators.h中，这里不再列举。

### 7.7.6 数组操作
#### 7.7.6.1 创建数组
创建一个新的HashTable分为两步：首先是分配zend_array内存，这个可以通过`ZVAL_NEW_ARR()`宏分配，也可以自己直接分配；然后初始化数组，通过`zend_hash_init()`宏完成，如果不进行初始化数组将无法使用。
```c
#define zend_hash_init(ht, nSize, pHashFunction, pDestructor, persistent) \
    _zend_hash_init((ht), (nSize), (pDestructor), (persistent) ZEND_FILE_LINE_CC)
```
* __ht：__ 数组地址HashTable*，如果内部使用可以直接通过emalloc分配
* __nSize：__ 初始化大小，只是参考值，这个值会被对齐到2^n，最小为8
* __pHashFunction：__ 无用，设置为NULL即可
* __pDestructor：__ 删除或更新数组元素时会调用这个函数对操作的元素进行处理，比如将一个字符串插入数组，字符串的refcount增加，删除时不是简单的将元素的Bucket删除就可以了，还需要对其refcount进行处理，这个函数就是进行清理工作的
* __persistent：__ 是否持久化

示例：
```c
zval        array;
uint32_t    size;

ZVAL_NEW_ARR(&array);
zend_hash_init(Z_ARRVAL(array), size, NULL, ZVAL_PTR_DTOR, 0);
```
#### 7.7.6.2 插入、更新元素
数组元素的插入、更新主要有三种情况：key为zend_string、key为普通字符串、key为数值索引，相关的宏及函数：
```c
// 1) key为zend_string

//插入或更新元素，会增加key的refcount
#define zend_hash_update(ht, key, pData) \
        _zend_hash_update(ht, key, pData ZEND_FILE_LINE_CC)

//插入或更新元素，当Bucket类型为indirect时，将pData更新至indirect的值，而不是更新Bucket
#define zend_hash_update_ind(ht, key, pData) \
        _zend_hash_update_ind(ht, key, pData ZEND_FILE_LINE_CC)

//添加元素，与zend_hash_update()类似，不同的地方在于如果元素已经存在则不会更新
#define zend_hash_add(ht, key, pData) \
        _zend_hash_add(ht, key, pData ZEND_FILE_LINE_CC)

//直接插入元素，不管key存在与否，如果存在也不覆盖原来元素，而是当做哈希冲突处理，所有会出现一个数组中key相同的情况，慎用!!!
#define zend_hash_add_new(ht, key, pData) \
        _zend_hash_add_new(ht, key, pData ZEND_FILE_LINE_CC)

// 2) key为普通字符串：char*

//与上面几个对应，这里的key为普通字符串，会自动生成zend_string的key
#define zend_hash_str_update(ht, key, len, pData) \
        _zend_hash_str_update(ht, key, len, pData ZEND_FILE_LINE_CC)
#define zend_hash_str_update_ind(ht, key, len, pData) \
        _zend_hash_str_update_ind(ht, key, len, pData ZEND_FILE_LINE_CC)
#define zend_hash_str_add(ht, key, len, pData) \
        _zend_hash_str_add(ht, key, len, pData ZEND_FILE_LINE_CC)
#define zend_hash_str_add_new(ht, key, len, pData) \
        _zend_hash_str_add_new(ht, key, len, pData ZEND_FILE_LINE_CC)

// 3) key为数值索引

//插入元素，h为数值
#define zend_hash_index_add(ht, h, pData) \
        _zend_hash_index_add(ht, h, pData ZEND_FILE_LINE_CC)

//与zend_hash_add_new()类似
#define zend_hash_index_add_new(ht, h, pData) \
        _zend_hash_index_add_new(ht, h, pData ZEND_FILE_LINE_CC)

//更新第h个元素
#define zend_hash_index_update(ht, h, pData) \
        _zend_hash_index_update(ht, h, pData ZEND_FILE_LINE_CC)

//使用自动索引值
#define zend_hash_next_index_insert(ht, pData) \
        _zend_hash_next_index_insert(ht, pData ZEND_FILE_LINE_CC)

#define zend_hash_next_index_insert_new(ht, pData) \
        _zend_hash_next_index_insert_new(ht, pData ZEND_FILE_LINE_CC)
```
#### 7.7.6.3 查找元素
```c
//根据zend_string key查找数组元素
ZEND_API zval* ZEND_FASTCALL zend_hash_find(const HashTable *ht, zend_string *key);

//根据普通字符串key查找元素
ZEND_API zval* ZEND_FASTCALL zend_hash_str_find(const HashTable *ht, const char *key, size_t len);

//获取数值索引元素
ZEND_API zval* ZEND_FASTCALL zend_hash_index_find(const HashTable *ht, zend_ulong h);

//判断元素是否存在
ZEND_API zend_bool ZEND_FASTCALL zend_hash_exists(const HashTable *ht, zend_string *key);
ZEND_API zend_bool ZEND_FASTCALL zend_hash_str_exists(const HashTable *ht, const char *str, size_t len);
ZEND_API zend_bool ZEND_FASTCALL zend_hash_index_exists(const HashTable *ht, zend_ulong h);

//获取数组元素数
#define zend_hash_num_elements(ht) \
    (ht)->nNumOfElements
//与zend_hash_num_elements()类似，会有一些特殊处理
ZEND_API uint32_t zend_array_count(HashTable *ht);
```
#### 7.7.6.4 删除元素
```c
//删除key
ZEND_API int ZEND_FASTCALL zend_hash_del(HashTable *ht, zend_string *key);

//与zend_hash_del()类似，不同地方是如果元素类型为indirect则同时销毁indirect的值
ZEND_API int ZEND_FASTCALL zend_hash_del_ind(HashTable *ht, zend_string *key);
ZEND_API int ZEND_FASTCALL zend_hash_str_del(HashTable *ht, const char *key, size_t len);
ZEND_API int ZEND_FASTCALL zend_hash_str_del_ind(HashTable *ht, const char *key, size_t len);
ZEND_API int ZEND_FASTCALL zend_hash_index_del(HashTable *ht, zend_ulong h);
ZEND_API void ZEND_FASTCALL zend_hash_del_bucket(HashTable *ht, Bucket *p);
```
#### 7.7.6.5 遍历
数组遍历类似foreach的用法，在扩展中可以通过如下的方式遍历：
```c
zval *val;
ZEND_HASH_FOREACH_VAL(ht, val) {
    ...
} ZEND_HASH_FOREACH_END();
```
遍历过程中会把数组元素赋值给val，除了上面这个宏还有很多其他用于遍历的宏，这里列几个比较常用的：
```c
//遍历获取所有的数值索引
#define ZEND_HASH_FOREACH_NUM_KEY(ht, _h) \
    ZEND_HASH_FOREACH(ht, 0); \
    _h = _p->h;

//遍历获取所有的key
#define ZEND_HASH_FOREACH_STR_KEY(ht, _key) \
    ZEND_HASH_FOREACH(ht, 0); \
    _key = _p->key;

//上面两个的聚合
#define ZEND_HASH_FOREACH_KEY(ht, _h, _key) \
    ZEND_HASH_FOREACH(ht, 0); \
    _h = _p->h; \
    _key = _p->key;

//遍历获取数值索引key及value
#define ZEND_HASH_FOREACH_NUM_KEY_VAL(ht, _h, _val) \
    ZEND_HASH_FOREACH(ht, 0); \
    _h = _p->h; \
    _val = _z;

//遍历获取key及value
#define ZEND_HASH_FOREACH_STR_KEY_VAL(ht, _key, _val) \
    ZEND_HASH_FOREACH(ht, 0); \
    _key = _p->key; \
    _val = _z;

#define ZEND_HASH_FOREACH_KEY_VAL(ht, _h, _key, _val) \
    ZEND_HASH_FOREACH(ht, 0); \
    _h = _p->h; \
    _key = _p->key; \
    _val = _z;
```
#### 7.7.6.6 其它操作
```c
//合并两个数组，将source合并到target，overwrite为元素冲突时是否覆盖
#define zend_hash_merge(target, source, pCopyConstructor, overwrite)                    \
    _zend_hash_merge(target, source, pCopyConstructor, overwrite ZEND_FILE_LINE_CC)

//导出数组
ZEND_API HashTable* ZEND_FASTCALL zend_array_dup(HashTable *source);
```
```c
#define zend_hash_sort(ht, compare_func, renumber) \
    zend_hash_sort_ex(ht, zend_sort, compare_func, renumber)
```
数组排序，compare_func为typedef int  (*compare_func_t)(const void *, const void *)，需要自己定义比较函数，参数类型为Bucket*，renumber表示是否更改键值，如果为1则会在排序后重新生成各元素的h。PHP中的sort()、rsort()、ksort()等都是基于这个函数实现的。

#### 7.7.6.7 销毁数组
```c
ZEND_API void ZEND_FASTCALL zend_array_destroy(HashTable *ht);
```
