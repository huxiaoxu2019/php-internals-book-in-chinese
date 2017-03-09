# 内存管理
`zval`结构体有两种作用：第一种，正如我们前文提到的那样，可以用来存储值及其类型。第二种，我们将会在本节讨论，它能够进行高效地管理变量的内存。

接下来我们来了解下引用计数（reference-counting）和复制写（copy-on-write）的概念，并弄清楚如何在扩展开发时使用它们。

## 值语义和引用语义（value- and reference-semantics）
在PHP中，所有变量都是值语义的，除非显示地使用引用。也就是说，无论是将一个变量传给一个函数还是给另一个变量赋值，这总是通过创建一个独立的拷贝实现的。举个例子：

```c
<?php

$a = 1;
$b = $a;
$a++;

// Only $a was incremented, $b stays as is:
var_dump($a, $b); // int(2), int(1)

function inc($n) {
    $n++;
}

$c = 1;
inc($c);

// The $c value outside the function and the $n inside the function are distinct
var_dump($c); // int(1)
```

上面的例子显然说明了记住这一特性是非常关键的。这一特性同样适用于对象。

```c
<?php

$obj = (object) ['value' => 1];

function fnByVal($val) {
    $val = 100;
}

function fnByRef(&$ref) {
    $ref = 100;
}

// The by-value function does not modify $obj, the by-reference function does:

fnByVal($obj);
var_dump($obj); // stdClass(value => 1)
fnByRef($obj);
var_dump($obj); // int(100)
```

有人会说在PHP5以后，对象会自动地以引用的形式进行传递，但上面的例子并非如此：一个值传递（by-value）的函数是不可以直接修改参数它本身的值的，只有接收引用参数的函数可以。

当然，上面的对象却是和引用传递有类似之处：尽管你不可以给它赋值一个完全不同的值，但是仍然可以改变这个对象的一些属性。这是因为对象的值其实是一个ID，这个ID可以用来寻找对象真实存储的值。值引用只是不允许你改变这个ID成其它不同的对象的ID，或者改成其它完全不同的变量，但是并不是不可以通过这个ID来改变对象内部的真实的值。


