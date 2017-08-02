# 扩展设计

在本章节，你将会学到如何设计PHP扩展。还会学习PHP生命周期，如何并且何时管理内存，可以使用的不同的钩子（hooks），可以用来替换的函数指针，进而来改变PHP内部的机制。你仍可以通过扩展的形式，设计并提供PHP函数或者类库，还可以通过INI文件来配置。

内容：

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
