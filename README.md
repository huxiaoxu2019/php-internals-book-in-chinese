# PHP Internals Book In Chinese
你看到的是PHP内核中文版，翻译自[PHP Internals Book](http://www.phpinternalsbook.com/index.html)。

## 为什么要翻译
 - 对技术的饥渴
 - 对英语的热爱
 - 方便汉语作为母语的人学习交流

## 官方网站
[PHP Internals Book](http://www.phpinternalsbook.com/index.html)

## 关于作者
[GenialX](http://blog.ihuxu.com/about-me)

## 内容目录
### PHP 5
 - Introduction
 - Using the PHP build system
   - Building PHP
   - Building PHP extensions
 - Creating PHP extensions
 - [Zvals](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals.md)
    - [基础结构](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/basic_structure.md) 
    - [内存管理](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/zvals/memory_management.md) 
    - Casts and operations
 - Implementing functions
 - Hashtables
   - Basic structure
   - Hashtable API
   - Symtable and array API
   - Hash algorithm and collisions
 - Classes and objects
   - Simple classes
   - Custom object storage
   - Implementing typed arrays
   - Object handlers
   - Iterators
   - Serialization
   - Magic interfaces - Comparable
   - Internal structures and implementation

### PHP 7

This part concerns only the PHP 7 branch. It is under development.

 - Introduction
 - Using the PHP build system
   - Building PHP
     - Why not use packages?
     - Obtaining the source code
     - Build overview
     - The ./buildconf script
     - The ./configure script
     - make and make install
     - Running the test suite
     - Fixing compilation problems and make clean
  - Building PHP extensions
    - Loading shared extensions
    - Installing extensions from PECL
    - Adding extensions to the PHP source tree
    - Building extensions using phpize
    - Displaying information about extensions
    - Extensions API compatibility
 - Internal types
   - Zvals
     - Basic structure
   - Strings management
     - Strings management: zend_string
     - smart_str API
     - PHP’s custom printf functions
   - The Resource type: zend_resource
     - What is the “Resource” type?
     - Resource types and resource destruction
     - Playing with resources
     - Reference counting resources
     - Persistent resources
   - HashTables: zend_array
   - Functions: zend_function
 - Extensions design
   - Learning the PHP lifecycle
     - The parallelism models
     - The PHP extensions hooks
     - Hooking by overwritting function pointers
   - A look into a PHP extension and extension skeleton
     - How the engine loads extensions
     - What is a PHP extension ?
     - Generating extension skeleton with scripts
     - Publishing API
   - Zend Extensions
     - On differences between PHP and Zend extensions
     - What is a Zend extension ?
     - Why need a Zend extension ?
     - API versions and conflicts management
     - Zend extensions lifetime hooks
     - Practice : my first example Zend extension
     - Hybrid extensions
   - Managing global state
   - Declaring and using INI settings
   - Registering and using PHP functions
     - zend_function_entry structure
     - Registering PHP functions
     - Declaring function arguments
     - The PHP function structure and API, in C
     - Adding tests
     - Playing with constants
     - A go with Hashtables (PHP arrays)
     - Managing references
 - Memory management
   - Zend Memory Manager
     - The two main kind of dynamic memory pools in PHP
     - Zend Memory Manager API
     - Zend Memory Manager debugging shields
     - ZendMM internal design
     - Common errors and mistakes
   - Debugging memory
     - A quick note about valgrind
     - Before starting
     - Memory leak detection example
     - Buffer overflow/underflow detection
     - Conclusions
 - [Zend engine](https://github.com/GenialX/php-internals-book-in-chinese/blob/master/php7/zend_engine.md) 
   - Zend Compiler
   - Zend Executor
   - Zend OPCache
