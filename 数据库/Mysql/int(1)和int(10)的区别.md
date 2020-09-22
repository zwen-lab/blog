### int(1) 和int(10)的区别

int类型括号后面的数字和能够存储的值无关，也就是说int(1)和int(10)存储的范围是一样的。

但是int(1)和int(10)在某些情情况下显示宽度不一样，int在无符号并且zerofill 的情况下，如果显示的宽度不够会在数字的左边填充0。

#### 测试一下

建一个表进行测试

```sql
CREATE TABLE `test_int` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `int1` int(1) unsigned zerofill DEFAULT NULL,
  `int3` int(3) unsigned zerofill DEFAULT NULL,
  `int10` int(10) unsigned zerofill DEFAULT NULL,
  `int10_a` int(10) unsigned DEFAULT '0',
  `int10_b` int(10) unsigned zerofill DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

#插入数据
INSERT INTO `test_int` (`id`, `int1`, `int3`, `int10`, `int10_a`, `int10_b`) VALUES ('1', '2', '3', '4', '5', '6');
INSERT INTO `test_int` (`id`, `int1`, `int3`, `int10`, `int10_a`, `int10_b`) VALUES ('2', '22', '33', '44', '55', '66');
INSERT INTO `test_int` (`id`, `int1`, `int3`, `int10`, `int10_a`, `int10_b`) VALUES ('3', '222', '333', '444', '555', '666');
INSERT INTO `test_int` (`id`, `int1`, `int3`, `int10`, `int10_a`, `int10_b`) VALUES ('4', '2222', '3333', '4444', '5555', '6666');

```

通过navicat 查看，结果如下图

![](E:\wamp\www\blog\数据库\Mysql\int-1.png)

于是就有人说，int(1)和int(10)并不存在上面说的无符号填充零时显示位数不够时会填充零的区别，所以得出结论无任何区别。

这个其实是navicat显示的锅，填充的零没有展示出来，导致看起来像没有填充一样。

如果直接通过命令行查看，就能很方便的查看填充零的情况。

![int-2](E:\wamp\www\blog\数据库\Mysql\int-2.png)

