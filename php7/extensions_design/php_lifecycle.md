# PHP的生命周期

为了掌握PHP内核原理，每个人都需要掌握其中复杂的生命周期机制。概述如下：

PHP启动。如果是以`CLI`或`FPM`方式运行，那么将会调用C的`main()`函数。如果，作为一个模块运行在WEB服务器中，像apxs2 SAPI（Apache 2），PHP将会紧随Apache启动后启动，同时运行模块启动序列（PHP就是其中的一口模块）。启动，内部称<b>模块启动步骤</b>。缩写形式为<b>MINIT</b>。

一旦启动，PHP准备好去处理一个/多个请求。当我们谈及PHP CLI时，PHP仅处理一个请求：带运行的脚本。然而，当我们谈及一个WEB服务 - 可能是PHP-FPM形式或者WEB服务器 模块模式 - PHP可以连续地处理多个请求。这取决于你是如何配置服务器的：可以配置成在杀死并回收改进程前，可以处理无穷个还是指定数量的请求。当每次处理一个新的请求时，PHP会执行<b>一个请求启动步骤</b>。我们称作<b>RINIT</b>。

当处理请求时，会（可能）产生一些内容。在关闭请求时，同时也准备好处理另外一个。关闭请求称为<b>请求关闭步骤</b>。

在处理了X个请求之后（一个，一些或者上千个等等），PHP会自我关闭并结束。关闭PHP进程称为“模块关闭步骤”。我们称作<b>MSHUTDOWN</b>。

下图即可描述上述的步骤：

<img src="http://www.phpinternalsbook.com/_images/php_classic_lifetime.png"/>

## 并行模型

在CLI环境中，情况会很简单：一个PHP进程处理一个请求：他会启动一个独立的PHP脚本，然后结束。CLI环境是Web环境的一个特例，Web环境会负责得多。

为了并行处理多个PHP请求，需要以并行模式来运行。下面有两种用PHP实现的方式。

 - 基于进程
 - 基于线程
 
 如果使用基于进程模型的方式，那么系统将分配给每个进程一个独立的解锁器。这种模型在Unix系统中很普遍。每一个请求在都各种进程中处理。PHP-CLI，PHP-FPM和PHP-CGI就是使用的这种模式。
 
同样的，基于线程模型的情况下，通过线程库，使每一个线程中拥有一个独立的PHP解释器。这种模型主要用于微软系统中，当然Unixes系列系统也可以。此时，要求PHP和扩展以ZTS（线程安全）的模式构建。

进程模型：

<img src="http://www.phpinternalsbook.com/_images/php_lifetime_process.png"/>

线程模型：

<img src="http://www.phpinternalsbook.com/_images/php_lifetime_thread.png"/>

> 注意
>
> 作为一名扩展开发者，PHP多进程模块并不是你的菜。因为你将不得不处理你的模块如何在多线程环境中运行的问题，尤其是在Windows平台下。



