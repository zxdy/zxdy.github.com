之前做到一个大日志文件（size > 1G）解析的项目，在此记录下对于大文本解析方式的效率比较。不同方式的性能差别很大，那个项目的日志解析时间能从原来的超过36小时优化到只需要2分钟，awk功不可没。

## bash 比较
bash脚本中对于文本的读取主要有以下四种，尽管 AWK 具有完全属于其本身的语法，但在此我也把它归在一起：

```bash
#方法一
func1(){
    rm -f $2
    echo "$(date) start to read"
    start_time=$(date +%s)
    cat $1|while read Line
    do
        echo $Line >> $2
    done
    end_time=$(date +%s)
    echo "$(date) end to read"
    echo "cost: "$((end_time-start_time))"sec"
}

#方法二
func2(){
    rm -f $2
    echo "$(date) start to read"
    start_time=$(date +%s)
    while read Line
    do
        echo $Line >> $2
    done <$1
    end_time=$(date +%s)
    echo "$(date) end to read"
    echo "cost: "$((end_time-start_time))"sec"
}

#方法三
func3(){
    rm -f $2
    echo "$(date) start to read"
    start_time=$(date +%s)
    for Line in `cat $1`
    do
        echo $Line >> $2
    done
    end_time=$(date +%s)
    echo "$(date) end to read"
    echo "cost: "$((end_time-start_time))"sec"
}

#func4
func4(){
    rm -f $2
    echo "$(date) start to read"
    start_time=$(date +%s)
    awk '{print $0}' $1 > $2
    echo "$(date) end to read"
    echo "cost: "$((end_time-start_time))"sec"
}


source=$1
dest=$2

#比较结果：
echo "####cat read: "
func1 $source $dest
echo "####redirect read: "
func2 $source $dest
echo "####for read: "
func3 $source $dest
echo "####awk read: "
func4 $source $dest
```
结果:

cat read:

>Thu Jan 15 07:57:50 GMT 2015 start to read

>Thu Jan 15 07:58:33 GMT 2015 end to read

>cost: 43sec

redirect read:

>Thu Jan 15 07:58:33 GMT 2015 start to read

>Thu Jan 15 07:59:01 GMT 2015 end to read

>cost: 28sec

for read:

>Thu Jan 15 07:59:01 GMT 2015 start to read

>Thu Jan 15 08:00:00 GMT 2015 end to read

>cost: 59sec

awk read:

>Thu Jan 15 08:00:00 GMT 2015 start to read

>Thu Jan 15 08:00:00 GMT 2015 end to read

>cost: 0sec

从以上结果可以看出，awk的效率远超其他方法

## python 比较
python 有三种读取文件的方法：

 - read() 会将所有内容读入到一个字符串中
 - readline() 每次读取一行
 - readlines() 将所有内容按行读取，返回一个列表，列表中每个元素是一个字符串，一个字符串是一行内容

所以从效率上讲， read() 和readlines()会比readline()高，但是同时对内存的要求也比较高，需要能一次性将文件内容放入内存中。但是如果这个文件很大的话，就会影响到程序运行的速度，甚至会导致程序挂掉，此时分行读取或是设置buff_size会是个更好的选择
```python
#!/usr/bin/env python
import time
import os
def func1(source,dest):
    os.remove(dest)
    with open(source, 'r') as fr:
        content=fr.read()
    with open(dest,'w') as fw:
        fw.write(content)
def  func2(source,dest):
    os.remove(dest)
    fw=open(dest,'w')
    for line in open(source,'r'):
        fw.write(line)
    fw.close
if __name__ == '__main__':
    from timeit import Timer
    t1=Timer("func1('log','log1')","from __main__ import func1")
    t2=Timer("func2('log','log1')","from __main__ import func2")
    print "read once: "+str(t1.timeit(1))
    print "read line: "+str(t2.timeit(1))
```

40M文件5次处理时间：
>read once: 0.308089971542

>read line: 1.17492413521

1G文件首次处理时间：
>read once: 8.17146706581

>read line: 4.13687205315

1G文件5次处理时间：
>read once: 7.32681894302

>read line: 30.3610920906

有意思的是，虽然一次性读入内存效率比line by line读取的效率差，但是假如重复处理同一个文件，一次性读取的总体效率反而高，所以python应该做了类似于缓存的机制。所以当我们用python处理大文本文件的时候需要综合考虑服务器内存，文件处理次数来决定使用哪种方式。
