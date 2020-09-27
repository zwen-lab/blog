#### laravel 5.2.45 从php7.0 升级到php 7.4 遇到的问题记录如下



##### 1.Array and string offset access syntax with curly braces is deprecated. 

导致的原因是在php7.4 + 版本中 array或者string类型 通过array{0} 或者string{0} 访问元素的方式已经弃用，只能通过array[0] 或者string[0] 进行访问。

具体php 7.4 废弃的地方请看官方文档：[https://www.php.net/manual/zh/migration74.deprecated.php]()



##### 2.count(): Parameter must be an array or an object that implements Countable.

这是在php7.2时 count() 函数的参数如果不是array或者没有实现Countable 时会触发一个错误。在php7.2 之前如果不是array也没有实现Countable 的话，除了null 特殊返回0，其他都返回1。



##### 3."continue" targeting switch is equivalent to "break". Did you mean to use "continue 2"? 

这是php7.3时的一个不向下兼容导致的。

下面这种情况会触发一个警告，在php中这样写是break的意思。

```php
<?php
while ($foo) {
    switch ($bar) {
      case "baz":
         continue;
         // Warning: "continue" targeting switch is equivalent to
         //          "break". Did you mean to use "continue 2"?
   }
}
```

具体可以仔细看看php7.3的不向下兼容的地方： [https://www.php.net/manual/zh/migration73.incompatible.php]()



##### 4.Trying to access array offset on value of type float.

参考官方文档：7.4 版本的向后不兼容更改，非数组的数组样式访问，现在，尝试将 null，bool，int，float 或 resource 类型的值用作数组 ( 例如 $null[“key”] ) 会产生一个notice。

文档地址:[https://www.php.net/manual/zh/migration74.incompatible.php]()



##### 总结

升级之前最好先看文档不向后兼容和废弃的功能

升级最好是保持逐级升级不要跨版本升级