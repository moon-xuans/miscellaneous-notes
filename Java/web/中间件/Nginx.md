# Nginx

## 1.Nginx的简介

### 1.1.什么是nginx？

Nginx 是高性能的 HTTP 和反向代理的服务器，处理高并发能力是十分强大的，能经受高负载的考验,有报告表明能支持高达 50,000 个并发连接数。

### 1.2.正向代理

需要在客户端配置代理服务器进行指定网站访问。

![image-20220320155427400](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320155427400.png)

### 1.3.反向代理

暴露的是代理服务器地址，隐藏了真实服务器的IP地址。

![image-20220320155557127](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320155557127.png)

### 1.4.负载均衡

增加服务器的数量，然后将请求分发到各个服务器上，将原先请求集中到单个服务器上的情况改为将请求分到多个服务器上，将负载分发到不同的服务器，也就是我们所说的负载均衡。

![image-20220320155908256](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320155908256.png)

### 1.5.动静分离

![image-20220320160317072](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320160317072.png)

## 2.Nginx的安装

### 2.1.准备工作

(1)打开虚拟机，使用远程连接工具连接linux操作系统

(2)到nginx官网下载软件**http://nginx.org/** 

![image-20220320161011231](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320161011231.png)

### 2.2.开始进行nginx安装

#### 2.2.1.安装pcre依赖

**第一步 联网下载pcre压缩文件依赖**

wget http://downloads.sourceforge.net/project/pcre/pcre/8.37/pcre-8.37.tar.gz

![image-20220320161229616](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320161229616.png)

**第二步 解压压缩文件**

使用命令tar –xvf pcre-8.37.tar.gz

第三步./configure 完成后，回到pcre目录下执行 make，最后执行make install

![image-20220320161413967](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320161413967.png)

#### 2.2.2.安装openssl、zlib、gcc依赖

yum -y install make zlib zlib-devel gcc-c++ libtool openssl openssl-devel

#### 2.2.3.安装nginx

使用命令解压

./configure

make && make install

进入目录/usr/local/nginx/sbin/nginx启动服务

![image-20220320162317277](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320162317277.png)

在windows系统中访问linux中nginx，默认不能访问的，因为防火墙问题

（1）关闭防火墙
（2）开放访问的端口号，80端口



**查看开放的端口号**

firewall-cmd --list-all

**设置开放的端口号**

firewall-cmd --add-service=http --permanent

firewall-cmd --add-port=80/tcp --permanent

**重启防火墙**

firewall-cmd --reload

![image-20220320162709868](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320162709868.png)

![image-20220320210225587](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320210225587.png)

## 3.Nginx的常用命令

进入nginx目录中

cd /usr/local/nginx/sbin

**1、查看nginx版本号**

./nginx -v

![image-20220320162829066](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320162829066.png)

**2、启动nginx**

./nginx

**3、停止nginx**

./nginx -s stop

**4、重新加载nginx**

./nginx -s reload

## 4.Nginx的配置文件

### 4.1.nginx配置文件位置

cd /usr/local/nginx/conf/nginx.conf

![image-20220320163612595](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320163612595.png)

### 4.2.配置文件中的内容

包含三部分内容

**（1）全局块**：配置服务器整体运行的配置指令

比如worker_processes 1;处理并发数的配置

**（2）events块**：影响Nginx服务器与用户的网络连接

比如worker_connections 1024;支持的最大连接数为1024

**（3）http块**

还包含两部分：

http全局块

server块 

## 5.Nginx配置实例

### 5.1.反向代理实例1

**1、实现效果**

（1）打开浏览器，在浏览器地址栏输入地址www.123.com，跳转到liunx系统tomcat主页面中

**2、准备工作**

（1）在liunx系统安装tomcat，使用默认端口8080

 tomcat安装文件放到liunx系统中，解压

 进入tomcat的bin目录中， ./startup.sh启动tomcat服务器

（2）对外开放访问的端口

firewall-cmd --add-port=8080/tcp --permanent

firewall-cmd --reload

查看已经开放的端口号

firewall-cmd --list-all

（3）在windows系统中通过浏览器访问tomcat服务器

![image-20220320205024750](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320205024750.png)

**3.访问过程的分析**

![image-20220320205123508](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320205123508.png)

**4.具体配置**

**第一步 在windows系统的host文件进行域名和ip对应关系的配置**

![image-20220320205308400](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320205308400.png)

![image-20220320205446720](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320205446720.png)

**第二步 在nginx进行请求转发的配置（反向代理配置）**

![image-20220320205836576](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320205836576.png)

**5、最终测试**

![image-20220320205930959](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320205930959.png)

### 5.2.反向代理实例2

**1、实现效果**

使用nginx反向代理，根据访问的路径跳转到不同端口的服务中

nginx监听端口为9001，

访问http://192.168.61.100:9001/edu/直接跳转到127.0.0.1:8080

访问http://192.168.61.100:9001/vod/直接跳转到127.0.0.1:8081

**2、准备工作**

（1）准备两个tomcat服务器，一个8080端口，一个8081端口(这里要记住打开8081的端口号)

（2）创建文件夹和测试页面

3、具体配置

（1）找到nginx配置文件，进行反向代理配置

![image-20220320211243238](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320211243238.png)

（2）开放对外访问的端口号9001 8080 8081

**4、最终测试**

![image-20220320215217277](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320215217277.png)

![image-20220320215249180](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320215249180.png)

### 5.3.负载均衡

**1、实现效果**

（1）浏览器地址栏输入地址http://192.168.65.100/edu/a.html，负载均衡效果，平均8080

和8081端口中

**2、准备工作**

（1）准备两台tomcat服务器，一台8080，一台8081

（2）在两台tomcat里面webapps目录中，创建名称是edu文件夹，在edu文件夹中创建页面a.html，用于测试

**3、在nginx的配置文件中进行负载均衡的配置**

![image-20220320220437726](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320220437726.png)

**4、nginx分配服务器策略**

**第一种轮询（默认）**

每个请求按时间顺序逐一分配到不同的后端服务器，如果后端服务器down掉，能自动剔除。

**第二种weight**

weight代表权重默认为1,权重越高被分配的客户端越多

**第三种ip_hash**

每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器

**第四种fair（第三方）**

按后端服务器的响应时间来分配请求，响应时间短的优先分配。

### 5.4.动静分离

**1、什么是动静分离**

![image-20220320224737725](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320224737725.png)

通过location指定不同的后缀名实现不同的请求转发。通过expires参数设置，可以使浏览器缓存过期时间，减少与服务器之前的请求和流量。具体expires定义：是给一个资源设定一个过期时间，也就是说无需去服务端验证，直接通过浏览器自身确认是否过期即可，所以不会产生额外的流量。此种方法非常适合不经常变动的资源。（如果经常更新的文件，不建议使用Expires来缓存），我这里设置3d，表示在这3天之内访问这个URL，发送一个请求，比对服务器该文件最后更新时间没有变化，则不会从服务器抓取，返回状态码304，如果有修改，则直接从服务器重新下载，返回状态码200。

**2、准备工作**

（1）在liunx系统中准备静态资源，用于进行访问

![image-20220320225740589](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320225740589.png)

**3、具体配置**

（1）在nginx配置文件中进行配置

![image-20220320231117347](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320231117347.png)

**4、最终测试**

（1）浏览器中输入地址

http://192.168.65.100/image/月亮.png

![image-20220320230631813](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320230631813.png)

因为配置文件autoindex on

![image-20220320230802927](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320230802927.png)

（2）在浏览器地址栏输入地址

http://192.168.65.100/www/a.html

![image-20220320230925029](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320230925029.png)

### 5.5.配置高可用的集群

**1、什么是nginx高可用**

![image-20220320231325710](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320231325710.png)

（1）需要两台nginx服务器

（2）需要keepalived

（3）需要虚拟ip

**2、配置高可用的准备工作**

（1）需要两台服务器192.168.65.100和192.168.65.102

（2）在两台服务器安装nginx

（3）在两台服务器安装keepalived

**3、在两台服务器安装keepalived**

（1）使用yum命令进行安装

yum install keepalived –y

![image-20220320233301499](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320233301499.png)

（2）安装之后，在etc里面生成目录keepalived，有文件keepalived.conf 

**4、完成高可用配置（主从配置）**

（1）修改/etc/keepalived/keepalivec.conf配置文件
```conf

global_defs {

 notification_email {

     acassen@firewall.loc

     failover@firewall.loc

     sysadmin@firewall.loc

 }

 notification_email_from Alexandre.Cassen@firewall.loc

 smtp_server 192.168.65.100

 smtp_connect_timeout 30

 router_id LVS_DEVEL # 这个是可以配置的，主机对应的名称

}

vrrp_script chk_http_port {

     script "/usr/local/src/nginx_check.sh"

     interval 2 #（检测脚本执行的间隔）

     weight 2

}

vrrp_instance VI_1 {

     state BACKUP # 备份服务器上将 MASTER 改为 BACKUP 

     interface ens33 //网卡

     virtual_router_id 51 # 主、备机的 virtual_router_id 必须相同

     priority 90 # 主、备机取不同的优先级，主机值较大，备份机值较小

     advert_int 1  
 

    authentication {

    auth_type PASS

    auth_pass 1111

    }

     virtual_ipaddress {

     192.168.65.50 // VRRP H 虚拟地址

     } 

}
```
（2）在/usr/local/src添加检测脚本
```shell

\#!/bin/bash

A=`ps -C nginx –no-header |wc -l`

if [ $A -eq 0 ];then

 /usr/local/nginx/sbin/nginx

 sleep 2

 if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then

 killall keepalived

 fi

fi
```

（3）把两台服务器上nginx和keepalived启动

启动nginx：./nginx

启动keepalived：systemctl start keepalived.service

**5、最终测试**

（1）在浏览器地址栏输入 虚拟ip地址192.168.65.50

（2）把主服务器（192.168.65.100）nginx 和 keepalived 停止，再输入 192.168.65.102

## 6.Nginx的原理

**1、master和worker**

![image-20220320233859468](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320233859468.png)

![image-20220320233923286](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320233923286.png)

**2、worker 如何进行工作的**

![image-20220320233949382](https://moon-axuan.oss-cn-beijing.aliyuncs.com/axuan/images/typora/image-20220320233949382.png)

**3、一个 master 和多个 worker 有好处**
（1）可以使用 nginx –s reload 热部署，利用 nginx 进行热部署操作
（2）每个 woker 是独立的进程，如果有其中的一个 woker 出现问题，其他 woker 独立的，
继续进行争抢，实现请求过程，不会造成服务中断
**4、设置多少个 worker 合适**
worker 数和服务器的 cpu 数相等是最为适宜的

**5、连接数 worker_connection**
第一个：发送请求，占用了 woker 的几个连接数？
答案：2 或者 4 个
第二个：nginx 有一个 master，有四个 worker，每个 woker 支持最大的连接数 1024，支持的
最大并发数是多少？

- 普通的静态访问最大并发数是： worker_connections * worker_processes /2。
- 而如果是 HTTP 作 为反向代理来说，最大并发数量应该是 worker_connections * 
  worker_processes/4。

