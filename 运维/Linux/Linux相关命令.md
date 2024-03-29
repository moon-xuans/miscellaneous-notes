# Linux相关命令

## 1.命令集合

### 1.1.整机：top，查看整机系统性能

![image-20220325232349675](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220325232349675.png)

使用top命令的话，重点关注的是%CPU、%MEM、load average三个指标

- load adverge三个指标:分别代表1、5、15分钟的负载情况

在这个命令下，按1的话，可以看到每个CPU的占用情况

uptime:系统性能命令的精简版

### 1.2.CPU:vmstat

- 查看CPU(包含但是不限于)
- 查看额外
- - 查看所有CPU核信息:mpstat -p ALL 2
  - 每个进程使用CPU的用量分解信息:pidstat -u 1 -p 进程编号

命令格式:`vmstat -n 2 3`

![image-20220325233308180](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220325233308180.png)

一般vmstat工具的使用是通过两个数字参数来完成的，第一个参数是采样的时间间隔数(单位秒),第二个参数是采样的次数。

**procs**

r: 运行和等待的CPU时间片的进程数，原则上1核的CPU的运行队列不要超过2，整个系统的运行队列不超过总核数的2倍，否则代表系统压力过大。如图，发现小于4，因此压力不大。

b: 等待资源的进程数，比如正在等待磁盘I/O、网络I/O等

**cpu**

us:用户进程消耗CPU时间百分比，us值高，用户进程消耗CPU时间多，如果长期大于50%，优化程序

sy:内核进程消耗的CPU时间百分比

![image-20220325234523894](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220325234523894.png)

us+sy参考值为80%，如果us+sy大于80%，说明可能存在CPU不足，从上面的图片可以看出，us+sy还没有超过80%，因此说明这台服务器的CPU消耗不是很高

id:处于空闲的CPU百分比

wa:系统等待IO的CPU时间百分比

st:来自于一个虚拟机偷取的CPU时间比

### 1.3.内存:free

- 应用程序可用内存数:free -m
- 应用程序可用内存/系统物理内存 > 70% 内存充足
- 应用程序可用内存/系统物理内存 < 20%，需要增加内存
- 20% < 应用程序可用内存/系统物理内存 < 70%，表示内存基本够用

free -h:以人类能看懂的方式查看物理内存

![image-20220326100713585](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326100713585.png)

free -m:以MB为单位，查看物理内存

![image-20220326100853265](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326100853265.png)

free -g:以GB为单位，查看物理内存

### 1.4.硬盘:df

 格式:`df -h`(-h:human，表示以人类能看到的方式换算)

![image-20220326101348194](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326101348194.png)



- 硬盘IO:iostat

系统慢有两种原因引起的，一个是CPU高，一个是大量IO操作

格式:`iostat -xdk 2 3`

![image-20220326102820533](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326102820533.png)

磁盘块设备分布

rkB/s:每秒读取数据量kB;

wkB/s:每秒写入数据量kB;

svctm I/O:请求的平均服务时间，单位毫秒

await I/O:请求的平均等待时间，单位毫秒，值越小，性能越好

util:一秒钟有百分之几的时间用于I/O操作。接近100%时，表示磁盘带宽跑满，需要优化程序或者增加磁盘；

rkB/s,wkB/s根据系统应用不同会有不同的值，但有规律遵循：长期、超大数据读写，肯定不正常，需要优化程序读取。

svctm的值与await的值很接近，表示几乎没有I/O等待，磁盘性能好，如果await的值远高于svctm的值，则表示I/O队列等待太长，需要优化程序或更换更快磁盘。

### 1.5.网络IO:ifstat

- 默认本地没有，下载ifstat

![image-20220326103928892](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326103928892.png)

## 2.生产环境服务器变慢，诊断思路和性能评估

记一次印象深刻的故障？

结合Linux和JDK命令一起分析，步骤如下:

- 使用top命令找出CPU占比最高的

![image-20220326111736045](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326111736045.png)

- ps -ef 或者 jps进一步定位，得到是一个怎么样的后台程序出的问题(jps是jdk提供的一个查看当前java进程的小工具， 可以看做是JavaVirtual Machine Process Status Tool的缩写。非常简单实用。)

![image-20220326112109256](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326112109256.png)

- 定位到具体线程或者代码
- - ps -mp 进程 -o THREAD,tid,time
  - 参数解释
  - - -m:显示所有的进程
    - -p:pid进程使用CPU的时间
    - -o:该参数后是用户自定义格式

![image-20220326112349015](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326112349015.png)

- 将需要的线程ID转换为16进制格式(英文小写格式)

- - printf "%x\n" 有问题的线程ID

![image-20220326112517798](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326112517798.png)

- jstack 进程ID | grep tid(16进制线程ID小写英文) -A60(jstack是java虚拟机自带的一种堆栈跟踪工具。)
- - 精准定位到错误的地方

![image-20220326112736829](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326112736829.png)

定位成功，确定是在该文件下的13行。

![image-20220326112814344](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220326112814344.png)