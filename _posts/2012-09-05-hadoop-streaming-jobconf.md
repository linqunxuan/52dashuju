---

title: HadoopStreaming开发要点以及jobconf常用配置参数
date: 2012-09-05
categories: hadoop

---

>Hadoop streaming是和hadoop一起发布的实用程序。它允许用户创建和执行使用任何程序或者脚本(例如: python,ruby,shell,php)编写的map或者reduce的mapreduce jobs。

### 一个简单的hadoop streaming可执行脚本

``` shell
$HADOOP_CMD jar $STREAM_JAR_PATH \
    -input $INPUT_FILE_PATH_1 \
    -output $OUTPUT_PATH \
    -mapper "python map.py" \
    -reducer "python reduce.py" \
    -jobconf "mapred.reduce.tasks=2" \ # reduce task的数量由mapred.reduce.tasks这个参数设定，默认值是1。
    -file ./map.py \
    -file ./reduce.py
```

- input: 指定作业的输入文件的HDFS路径，支持使用\*通配符，支持指定多个文件或目录，可多次使用

- output: 指定作业的输出文件的HDFS路径，路径必须存在，执行作业用户必须有创建该目录的权限，只能使用一次

- mapper: 用户自己写的mapper程序

- reducer: 用户自己写的reduce程序

- file: 打包文件到提交的作用中，

  （1）map和reduce的执行文件

  （2）map和reduce要用输入的文件，如配置文件类似的配置还有-cacheFile, -cacheArchive分别用于向计算节点分发HDFS文件和HDFS压缩文件

- jobconf: 提交作业的一些配置属性

在上面的例子中，mapper和reducer都是可执行程序，mapper的输入是读取stdin，reducer的输出是输出到stdout。

Hadoop Streaming可以创建mapreduce任务，提交任务到正确的cluster，监控任务的执行过程直到任务完成。

当mapper定义为可执行程序时，每个mapper task初始化都会独立启动该进程。

Mapper task运行时，将输入的文件转换成行来处理，然后传入到stdin；

同时，将输出处理为key-value，作为mapper的输出；

默认情况下，每行数据第一个tab前面的数据是key，剩下的作为value。

如果某行数据中没有tab，则将整行数据作为key，value值为null。然而，这些也可以自定义处理。

当reducer定义为可执行程序时，每个reducer task初始化都会独立启动该进程。

Reducer task运行时，将输入的key-value数据转换成行数据，作为reducer的输入。

同时，reducer收集行数据，将行数据转换成key-value形式输出。

默认情况下，每行数据第一个tab前面的数据是key，剩下的作为value。然而，这些也可以自定义处理。

这是mapreduce框架和hadoop streaming之间的基本通信协议。

### Jobconf常用配置参数

* mapred.map.tasks  map task数目
* mapred.reduce.tasks  reduce task数目
* stream.num.map.output.key.fields  指定map task输出记录中key所占的域数目
* num.key.fields.for.partition  指定对key分出来的前几部分做partition而不是整个key
* mapred.job.name   作业名
* mapred.job.priority  作业优先级
* mapred.job.map.capacity  最多同时运行map任务数
* mapred.job.reduce.capacity  最多同时运行reduce任务数
* mapred.task.timeout  任务没有响应（输入输出）的最大时间
* mapred.compress.map.output  map的输出是否压缩
* mapred.map.output.compression.codec  map的输出压缩方式
* mapred.output.compress  reduce的输出是否压缩
* mapred.output.compression.codec  reduce的输出压缩方式
* stream.map.output.field.separator  map输出分隔符

