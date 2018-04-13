---
title: Linux CPU负载告警规则——Top命令的load average划重点
date: 2012-10-05 19:44:12
categories: Linux运维
---

>Linux运维中,我们知道CPU会有负载过荷的情况,下面我来介绍一下这个负载告警的规则。

#### 使用top命令查看第一行中的load average

``` shell

top - 16:54:26 up 99 days, 19:07,  3 users,  load average: 3.85, 3.69, 3.42
#load average: 3.85, 3.69, 3.42 系统负载，即任务队列的平均长度。
#三个数值分别为 1分钟、5分钟、15分钟前到现在的平均值。

```

理论上服务器负载小于70%，则还能坚持运行下去。

公式如下:
``` shell
平均值/CPU核数 > 0.7  大于70%的负载,需要告警了

平均值/CPU核数 <= 0.7  这波稳了,还能撑住
```

查看cpu个数和核数的命令如下:

``` shell

# 总核数 = 物理CPU个数 X 每颗物理CPU的核数 
# 总逻辑CPU数 = 物理CPU个数 X 每颗物理CPU的核数 X 超线程数
# 查看物理CPU个数
cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l
# 查看每个物理CPU中core的个数(即核数)
cat /proc/cpuinfo| grep "cpu cores"| uniq
# 查看逻辑CPU的个数
cat /proc/cpuinfo| grep "processor"| wc -l

```

* 假设CPU个数为1,核数为4,总核数为4


* 则按照现有的5分钟平均值, 3.69/4 = 0.92 负载92%以上了，这时候就需要告警了。


### © 著作权归作者所有，转载需联系作者

### END