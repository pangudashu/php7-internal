## 4.5 include/require


调用文件的变量范围将全部被包含文件所继承，具体实现：

include文件在attach_symbol_table()时如果发现要插入的全局变量已经存在，则会将新的全局变量的value指向原value，然后将全局变量更新为新的全局变量。

include执行完成return时zend_detach_symbol_table()时会将所有的全局变量的value拷到EG(symbol_table)中(CV->symbol table)，原调用文件重新attach，这样会把include文件中的全局变量移到当前文件，如果include中修改了原文件的全局变量，此时也会将更新后的value重新赋给原文件的变量(symbol table -> CV)。
