## 5.1 Zend内存池
zend针对内存的操作封装了一层，实现了更高效率的内存利用，其实现主要参考了tcmalloc的设计。

zend内存池有两种粒度的内存块：chunk、page，每个chunk的大小为2M，page大小为4KB，一个chunk被切割为512个page，申请内存时按照三种情况处理：



