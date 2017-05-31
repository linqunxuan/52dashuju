---
title: 基于Hadoop1.2版本的MapReduce实现WordCount功能
category: MapReduce
categories: MapReduce
date: 2013-11-18
---


>
> 前提: 
> hadoop集群搭建好
> hadoop版本: 1.2.x
> python版本: 2.6.x或2.7.x
>


### 在master的根目录建立 learn_hadoop/python_wordcount 并进入目录

``` bash
[root@master learn_hadoop]# mkdir -p ~/learn_hadoop/python_wordcount
[root@master learn_hadoop]# cd ~/learn_hadoop/python_wordcount/
```


### 准备数据源
``` bash
cat /etc/passwd | tr ':' ' ' | tr '/' ' ' > test.txt
```

### shell实现简单的wordcount
``` bash
cat test.txt | tr ' ' '\n' | sort | uniq -c | sort -rn -k1 | head -10
```

### 编写map.py 用于数据的分割
``` python
#!/usr/local/bin/python
import sys
# 从标准输入流获取数据源 并遍历
for line in sys.stdin:
    # strip() 表示删除空白符  相当于 java的String.trim()
    # 每一行以空格为分割点 得到数组
	ss = line.strip().split(' ')
    # 遍历数组
	for s in ss:
        # 继续strip() 并判断字符串是否为""
		if s.strip() != "":
            # 输出 单词 和 1(表示次数)
			print "%s\t%s" % (s, 1)
```


### 编写reduce.py 用于数据的合并
``` python
#!/usr/local/bin/python
import sys
# 定义一个变量用于存放当前记录的key
current_word = None
# 定义一个数组用于存放获取到key的值统计数组
count_pool = 
# 定义一个变量用于存放单词出现的总次数
sum = 0
for line in sys.stdin:
	# 每一行记录都切割成键值对 
	# 例如: the	1 变为 {"the", 1}
	word, val = line.strip().split('\t')
	# 当前key 为 none
	if current_word == None:
		# 赋值
		current_word = word
	# 当前key != 此次获取到key
	if current_word != word:
		# 遍历统计数组 得到单词出现的总次数 并且打印出来  
		for count in count_pool:
			# 累计单词出现的总次数
			sum += count
		# 打印出来 当前单词 以及 总次数	
		print "%s\t%s" % (current_word, sum)
		# 将此次得到的单词 赋值给 当前key这个变量
		current_word = word
		# 初始化统计数组
		count_pool = 
		# 初始化总次数
		sum = 0
	# 将此次键值对的值追加到数组中
	count_pool.append(int(val))
# 最后一行记录 会循环统计并打印出 
# 当前单词 以及 出现次数
for count in count_pool:
	sum += count
print "%s\t%s" % (current_word, str(sum))
```

### 本地测试的命令 得到输出数据 按倒序排列单词 以及 出现次数
``` bash
cat test.txt | python map.py | sort -k1 | python reduce.py | sort -k2 -rn
```


### 编写run.sh 将脚本提交到hadoop运行
``` bash
vim run.sh

# 根据自己的实际hadoop安装目录定义变量
HADOOP_CMD="/usr/local/hadoop-1.2.1/bin/hadoop"
STREAM_JAR_PATH="/usr/local/hadoop-1.2.1/contrib/streaming/hadoop-streaming-1.2.1.jar"
INPUT_FILE_PATH_1="/test.txt"
OUTPUT_PATH="/output"
# hdfs上如果已存在/output目录则需要下面的命令
$HADOOP_CMD fs -rmr -skipTrash $OUTPUT_PATH
# 执行命令
$HADOOP_CMD jar $STREAM_JAR_PATH \
		-input $INPUT_FILE_PATH_1 \
		-output $OUTPUT_PATH \
		-mapper "python map.py" \
		-reducer "python reduce.py" \ 
		-file ./map.py \
		-file ./reduce.py
```

### 给run.sh赋予执行权限
``` bash
chmod -R 775 run.sh
```

### 执行
``` bash
./run.sh
```

### 查看hdfs上的输出文件
``` bash
[root@master python_wordcount]# hadoop fs -lsr /output
Warning: $HADOOP_HOME is deprecated.
-rw-r--r--   3 root supergroup          0 2013-11-15 01:28 /output/_SUCCESS
drwxr-xr-x   - root supergroup          0 2013-11-15 01:28 /output/_logs
drwxr-xr-x   - root supergroup          0 2013-11-15 01:28 /output/_logs/history
-rw-r--r--   3 root supergroup      16597 2013-11-15 01:28 /output/_logs/history/job_201311150126_0003_1492104524860_root_streamjob2185335538628447165.jar
-rw-r--r--   3 root supergroup      52682 2013-11-15 01:28 /output/_logs/history/job_201704110143_0003_conf.xml
-rw-r--r--   3 root supergroup        734 2013-11-15 01:28 /output/part-00000
```

### 发现hdfs上/output/part-00000 该文件为输出数据结果
#### 查看hdfs上的输出数据 按倒序 单词 以及 出现次数

``` bash
hadoop fs -text /output/part-00000 | sort -k2 -rn
```

#### 输出结果与本地测试结果一致，大功告成

## END


