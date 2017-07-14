# Zvals

本章节的主题为用来表达PHP变量的zval数据结构。我们将会围绕zvals的概念和如何在扩展开发中使用两方面来进行阐述。

目录
 - [基础结构](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/basic_structure.md)
   - [类型和值](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/basic_structure.md#user-content-类型和值)
   - [访问宏](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/basic_structure.md#user-content-访问宏)
   - [赋值](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/basic_structure.md#user-content-赋值)
 - [内存管理](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/memory_management.md)
   - [值语义和引用语义](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/memory_management.md#user-content-值语义和引用语义)
   - [引用计数和写时复制](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/memory_management.md#user-content-引用计数和写时复制)
   - [分配并初始化zvals](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/memory_management.md#user-content-分配并初始化zvals)
   - [管理引用计数和zval销毁](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/memory_management.md#user-content-管理引用计数和zval销毁)
   - [复制zvals](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/memory_management.md#user-content-复制zvals)
   - [分离zvals](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/memory_management.md#user-content-分离zvals)
 - 类型转换和操作符
   - 基础操作符
   - 比较
   - 类型转换
