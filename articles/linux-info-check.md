## 1. hostname：查看主机名

## 2. uname -a  查看所有信息
    -v/-r :查看内核版本
    -n  相当于查看主机名
    -s  查看内核名称
    -o  操作系统
    -i  查看硬件平台
    (1) lsb_release -a
    (2) cat /proc/version
    (3) cat /etc/issue
    (4) cat /etc/*release*

## 3. 查看cpu信息：

3.1 物理CPU个数：
   
    cat /proc/cpuinfo | grep "physical id" | sort -u | wc -l

3.2 逻辑CPU个数

    cat /proc/cpuinfo | grep "processor" | wc -l

3.3 CPU核数

    cat /proc/cpuinfo| grep "cpu cores"| uniq

3.4 判断CPU是否64位

    检查cpuinfo中的flags区段，看是否有lm标识。


##5. 查看内存

grep MemTotal /proc/meminfo 
free

##6. 查看交换分区大小

grep SwapTotal /proc/meminfo 

##7. 查看进程信息

ps -ef  
ps -aux

##8. 显示档案系统的最大空间及使用情况

df -h|-H
-ih 

##9. 查看文件和目录占用空间

du -a  显示全部目录和次目录下每个档案所占的空间
du -s  显示各档案大小的总和 

