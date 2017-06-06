---
title: MySQL server has gone away 问题的解决方法
categories: mysql
date: 2012-09-27
---

>将500M以上数据导入mysql，有时候会报:**MySQL server has gone away** 这个错误提示，那么该如何解决呐?


MySQL根据配置文件会限制server接受的数据包大小。

有时候大的插入和更新会被`max_allowed_packet`参数限制掉，导致失败。


#### 查看目前配置

``` shell
mysql> show global variables like 'max_allowed_packet';
+--------------------+-----------+
| Variable_name      | Value     |
+--------------------+-----------+
| max_allowed_packet | 4194304 |
+--------------------+-----------+
1 row in set, 1 warning (0.00 sec)

```

* value值4194304单位为字节(b)

#### 修改`max_allowed_packet`参数值为200M

``` shell
mysql> set global max_allowed_packet = 20*1024*1024*10;
```

再重新导入数据到mysql中，如果还是出现问题，继续调大`max_allowed_packet`参数值.

如果想永久更改`max_allowed_packet`参数值，则需要在my.cnf中进行修改.

### © 著作权归作者所有，转载需联系作者

### END
