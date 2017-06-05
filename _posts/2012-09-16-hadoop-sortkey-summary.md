---
title: HadoopStreaming开发中key排序和分桶小结
date: 2012-09-16
categories: hadoop

---

>[Hadoop](http://lib.csdn.net/base/hadoop)用于对key的排序和分桶的设置选项比较多和复杂，目前在公司内主要使用KeyFieldBasePartitioner和KeyFieldBaseComparator。

### 基本概念

Partition：分桶过程，用户输出的key经过partition分发到不同的reduce里，因而partitioner就是分桶器，一般用平台默认的hash分桶也可以自己指定。 
Key：是需要排序的字段，相同分桶&&相同key的行排序到一起。

下面以一个简单的文本作为例子，通过搭配不同的参数跑出真实作业的结果来演示这些参数的使用方法。 
假设map的输出是这样以点号分隔的若干行：

``` shell
d.1.5.23
e.9.4.5
e.5.9.22
e.5.1.45
e.5.1.23
a.7.2.6
f.8.3.3
```

我们知道，在streaming模式默认hadoop会把map输出的一行中遇到的第一个设定的字段分隔符前面的部分作为key，后面的作为value，如果输出的一行中没有指定的字段分隔符，则整行作为key，value被设置为空字符串。 那么对于上面的输出，如果想用map输出的前2个字段作为key，后面字段作为value，并且不使用hadoop默认的“\t”字段分隔符，而是根据该文本特点使用“.”来分割，需要如何设置呢?

``` shell
bin/hadoop streaming 
-input /tmp/comp-test.txt \
-output /tmp/xx \
-mapper cat -reducer cat \
-jobconf stream.num.map.output.key.fields=2 \
-jobconf stream.map.output.field.separator=. \
-jobconf mapred.reduce.tasks=5
```

#### 结果：

``` shell
e.9 4.5
f.8 3.3
——————
d.1 5.23
e.5 1.23
e.5 1.45
e.5 9.22
——————
a.7 2.6
```

#### 总结：

从结果可以看出，在reduce的输出中，前两列和后两列用“\t”分隔，证明map输出时确实把用“.”分隔的前两列作为key，后面的作为value。并且前两列相同的“e.5”开头的三行被分到了同一个reduce中，证明确实以前两列作为key整体做的partition。 

* stream.num.map.output.key.fields 设置map输出的前几个字段作为key 
* stream.map.output.field.separator 设置map输出的字段分隔符

### KeyFieldBasePartitioner的用法

如果想要灵活设置key中用于partition的字段，而不是把整个key都用来做partition。就需要使用hadoop中的org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner了。 
下面只用第一列作partition，但依然使用前两列作为key。

``` shell
bin/hadoop streaming -input /tmp/comp-test.txt -output /tmp/xx -mapper cat -reducer cat \
-partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
-jobconf stream.num.map.output.key.fields=2 \
-jobconf stream.map.output.field.separator=. \
-jobconf map.output.key.field.separator=. \
-jobconf num.key.fields.for.partition=1 \
-jobconf mapred.reduce.tasks=5
```

#### 结果:

``` shell
d.1 5.23
——————
e.5 1.23
e.5 1.45
e.5 9.22
e.9 4.5
——————
a.7 2.6
f.8 3.3
```

#### 总结：

从结果可以看出，这次“e”开头的行都被分到了一个桶内，证明做partition是以第一列为准的，而key依然是前两列。并且在同一个partition内，先按照第一列排序，第一列相同的，按照第二列排序。这里要注意的是使用map.output.key.field.separator来指定key内字段的分隔符，这个参数是KeyFieldBasePartitioner和KeyFieldBaseComparator所特有的。 

* map.output.key.field.separator 设置key内的字段分隔符
* num.key.fields.for.partition 设置key内前几个字段用来做partition

事实上`KeyFieldBasePartitioner`还有一个高级参数`mapred.text.key.partitioner.options`，

这个参数可以认为是`num.key.fields.for.partition`的升级版，它可以指定不仅限于key中的前几个字段用做partition，

而是可以单独指定key中某个字段或者某几个字段一起做partition。 

比如上面的需求用`mapred.text.key.partitioner.options`表示为 `mapred.text.key.partitioner.options=-k1,1` 

注意`mapred.text.key.partitioner.options`和`num.key.fields.for.partition`不需要一起使用，一起使用则以`num.key.fields.for.partition`为准。

这里再举一个例子，使用`mapred.text.key.partitioner.options`

``` shell
bin/hadoop streaming -input /tmp/comp-test.txt -output /tmp/xx -mapper cat -reducer cat \
-partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
-jobconf stream.num.map.output.key.fields=3 \
-jobconf stream.map.output.field.separator=. \
-jobconf map.output.key.field.separator=. \
-jobconf mapred.text.key.partitioner.options=-k2,3 \
-jobconf mapred.reduce.tasks=5
```

#### 结果:

``` shell
e.9.4   5
——————
a.7.2   6
e.5.9   22
——————
d.1.5   23
e.5.1   23
e.5.1   45
f.8.3   3
```

可见，这次是以前3列作为key的，而partition则以key中的第2-3列，因此以“e”开头的行被拆散了，但第二三列相同的“5，1”被分到一个桶内。在同一个桶内，依然是从key的第一列开始排序的，注意，KeyFieldBasePartitioner只影响分桶并不影响排序。 

* mapred.text.key.partitioner.options 设置key内某个字段或者某个字段范围用做partition

### KeyFieldBaseComparator的用法

首先简单解释一下hadoop框架中key的comparator，对于hadoop所识别的所有[Java](http://lib.csdn.net/base/java)的key类型（在框架看来key的类型只能是java的），很多类型都自定义了基于字节的比较器，比如Text，IntWritable等等，如果不特别指定比较器而使用这些类型默认的，则会将key作为一个整体的字节数组来进行比较。而KeyFieldBaseComparator则相当于是一个可以灵活设置比较位置的高级比较器，但是它并没有自己独有的比较逻辑，而是使用默认Text的基于字典序或者通过-n来基于数字比较。 
之前的例子使用KeyFieldBasePartitioner自定义了使用key中的部分字段做partition，现在我们通过**org.apache.hadoop.mapred.lib.KeyFieldBasedComparator**来自定义使用key中的部分字段做比较。

这次把前四列都作为key，前两列做partition，排序依据优先依据第三列正序(文本序)，第四列逆序(数字序)的组合排序。

``` shell
bin/hadoop streaming -input /tmpcomp-test.txt -output /tmp/xx -mapper cat -reducer cat \
-partitioner org.apache.hadoop.mapred.lib.KeyFieldBasedPartitioner \
-jobconf mapred.output.key.comparator.class=org.apache.hadoop.mapred.lib.KeyFieldBasedComparator \
-jobconf stream.num.map.output.key.fields=4 \
-jobconf stream.map.output.field.separator=. \
-jobconf map.output.key.field.separator=. \
-jobconf mapred.text.key.partitioner.options=-k1,2 \
-jobconf mapred.text.key.comparator.options="-k3,3 -k4nr" \
-jobconf mapred.reduce.tasks=5
```

#### 结果:

``` shell
e.5.1.45
e.5.1.23
d.1.5.23
e.5.9.22
——————
a.7.2.6
——————
f.8.3.3
e.9.4.5
```

#### 总结:

从结果可以看出，符合预期的按照先第三列文本正序，然后第四列基于数字逆序的排序。 
另外注意，如果这种写法 
`mapred.text.key.comparator.options="-k2"` 
则会从第二列开始，用字典序一直比较到key的最后一个字节。所以对于希望准确排序字段的需求，还是使用“k2,2”这种确定首尾范围的形式更好。另外如果给定的key中某些行需要排序的列数不够时，会比较到最后一列，缺列的行默认缺少的那一列排序值最小。 

* mapred.text.key.comparator.options 设置key中需要比较的字段或字节范围

### © 著作权归作者所有，转载需联系作者

### END