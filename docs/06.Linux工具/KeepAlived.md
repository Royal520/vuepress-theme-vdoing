---
title:  'keepalived'
date: 2020-05-12 15:09:57
permalink: /pages/linux/0602
sidebar: false
article: false
---
keepalive高可用软件配置及详解
===================

最新推荐文章于 2024-06-14 10:35:17 发布

![](https://csdnimg.cn/release/blogv2/dist/pc/img/original.png)

[小鱼儿&](https://blog.csdn.net/weixin_45625174 "小鱼儿&") ![](https://csdnimg.cn/release/blogv2/dist/pc/img/newCurrentTime2.png) 于 2020-09-16 13:28:01 发布

![](https://csdnimg.cn/release/blogv2/dist/pc/img/articleReadEyes2.png) 阅读量1.5w ![](https://csdnimg.cn/release/blogv2/dist/pc/img/tobarCollect2.png) ![](https://csdnimg.cn/release/blogv2/dist/pc/img/tobarCollectionActive2.png) 收藏 61

![](https://csdnimg.cn/release/blogv2/dist/pc/img/newHeart2023Active.png) ![](https://csdnimg.cn/release/blogv2/dist/pc/img/newHeart2023Black.png) 点赞数 11

文章标签： [linux](https://so.csdn.net/so/search/s.do?q=linux&t=all&o=vip&s=&l=&f=&viparticle=) [keepalive](https://so.csdn.net/so/search/s.do?q=keepalive&t=all&o=vip&s=&l=&f=&viparticle=)

版权声明：本文为博主原创文章，遵循 [CC 4.0 BY-SA](http://creativecommons.org/licenses/by-sa/4.0/) 版权协议，转载请附上原文出处链接和本声明。

本文链接：[https://blog.csdn.net/weixin\_45625174/article/details/108600061](https://blog.csdn.net/weixin_45625174/article/details/108600061)

版权



### [Keepalived](https://so.csdn.net/so/search?q=Keepalived&spm=1001.2101.3001.7020)高可用软件简介：

*   目前互联网主流的**实现WEB网站**及**数据库服务**高可用软件包括：**keepalived、heartbeat**等。**Heartbeat**是比较早期的实现高可用软件，而**keepalived**是**目前轻量级的管理方便**、**易用的高可用软件解决方案，得到互联网公司IT人的青睐**。
*   Keepalived是一个类似于工作在**layer 3, 4 & 7**交换机制的软件，Keepalived软件有两种功能，**分别是健康检查**、**VRRP冗余协议**，keepalived是模块化设计，不同模块负责不同的功能，keepalived常用模块包括：

> **Core**：是keepalived的核心，负责主进程的启动和维护，全局配置文件的加载解析等 。  
> **Check**：负责healthchecker(健康检查)，包括了各种健康检查方式，以及对应的配置的解析包括LVS的配置解析；  
> **Vrrp**：VRRPD子进程，VRRPD子进程就是来实现VRRP协议；  
> **Libipfwc**：iptables(ipchains)库，配置LVS会用到；  
> **Libipvs**：虚拟服务集群，配置LVS会使用。

*   **Keepalived的作用**是检测服务器的状态，**如果有一台web服务器、Mysql服务器宕机，或工作出现故障**，Keepalived将**检测到后**，会将有故障的web服务器或者Mysql服务器从**系统中剔除**，当服务器**工作正常**后Keepalived自动将web、Mysql服务器**加入到服务器群中**。
*   **这些工作全部自动完成**，**不需要人工干涉**，需要人工做的只是**修复故障的WEB和Mysql服务器**。Layer3,4&7工作在IP/TCP协议栈的**网络层、传输层及应用层**，实现原理分别如下：

> **Layer3**：Keepalived使用Layer3的方式工作式时，Keepalived会**定期向服务器群中的服务器发送一个ICMP的数据包**（如果发现某台服务的IP地址无法ping通，Keepalived便报告这台服务器失效，并将它从服务器集群中剔除。Layer3的方式是以服务器的IP地址是否有效作为服务器工作正常与否的标准。）  
> **Layer4**: Layer4主要以**TCP端口的状态来决定服务器工作正常与否**。如WEB server的服务端口一般是80，（如果Keepalived检测到80端口没有启动，则Keepalived将把这台服务器从服务器群中剔除）  
> **Layer7**：Layer7工作在应用层，Keepalived将**根据用户的设定检查服务器程序的运行是否正常**。（如果与用户的设定不相符，则Keepalived将把服务器从服务器群中剔除）

*   生产环境使用Keepalived正常运行，**共启动3个进程**，一个是**父进程，负责监控其子进程**，**一个是VRRP子进程**，另外一个是**Checkers子进程**。
*   **两个子进程都被系统Watchlog看管**，两个子进程各自负责自己的事，**Healthcheck子进程检查各自服务器的健康状况**，如果**Healthcheck进程检查**到**Master上服务不可用**了，就会**通知**本机上的**VRRP子进程**，让他**删除通告**，并且**去掉虚拟IP**，**转换为BACKUP**状态。

### 1\. Keepalived VRRP原理剖析一：

*   Virtual Router Redundancy Protocol（VRRP）技术，**虚拟路由器冗余协议**。VRRP由**IETF**提出，目的是为了解决局域网中配置**默认网关的单点失效问题**，**1998年**已推出正式的RFC2338协议标准。
*   VRRP广泛应用在边缘网络中，它的设计目标是支持特定情况下IP数据流量失败转移不会引起混乱，允许主机使用单路由器，以及及时在实际第一跳路由器使用失败的情形下仍能够维护路由器间的连通性。
*   在现实的网络环境中，两台需要通信的主机大多数情况下并没有直接的物理连接。对于这样的情况，它们之间路由怎样选择？主机如何选定到达目的主机的下一跳路由，这个问题通常的解决方法有二种：

> 1.  在主机上使用动态路由协议RIP、OSPF；
> 2.  在主机上配置静态路由（默认网关）

*   在主机上**配置路态路由是非常不切实际**的，因为管理、维护成本以及是否支持等诸多问题。**配置静态路由就变得十分流行**，但路由器(或者说默认网关default gateway)却经常成为**单点**，VRRP的目的就是为了**解决**静态路由**单点故障**问题。VRRP通过一竞选(election)协议来动态的将路由任务交给LAN中虚拟路由器中的某台VRRP路由器。

### 2\. Keepalived VRRP原理剖析二：

*   通过VRRP技术可以将**两台物理（路由器）主机当成路由器**，两台物理机主机组成一个**虚拟路由集群**，Master主机会产生**VIP地址**，该VIP地址负责**转发**用户发起的IP包或者负责处理用户的**请求**，**Nginx+Keepalived组合**，用户的**请求**直接**访问**keepalived **VIP地址**，然后访问Master相应**服务和端口**。
*   在**VRRP虚拟路由器集群**中，由**多台物理的路由器组成**，但是这多台的物理路由器并**不能同时工作**，而是由一台为**MASTER路由器**负责路由**工作**，其它的都是BACKUP，MASTER并非一成不变，VRRP会让每个VRRP路由器参与竞选，最终获胜的就是MASTER。
*   **MASTER**拥有一些**特权**，例如拥有虚拟**路由器的IP地址**或者成为VIP，拥有特权的MASTER要**负责转发**发送给网关地址的包和响应ARP请求。
*   **VRRP通过竞选**协议来实现虚拟路由器的功能，所有的协议报文都是通过**IP组播**(multicast)包(组播地址224.0.0.18)形式发送的。虚拟路由器由VRID(范围0-255)和一组IP地址组成，对外表现为一个周知的MAC地址。所以在一组虚拟路由器集群中，不管谁是MASTER，对外都是相同的**MAC和VIP**。**客户端主机**并**不需要**因为MASTER的改变而**修改**自己的路由配置。
*   作为MASTER的VRRP路由器会一直**发送**VRRP**组播包**(VRRP Advertisement message)，BACKUP**不会抢占**MASTER，除非它的**优先级**(Priority)更高。当MASTER不可用时(BACKUP收不到组播包时)， 多台BACKUP中**优先级**最高的这台会**抢占**为MASTER。这种抢占是非常快速的，以保证服务的**连续性**。由于安全性考虑VRRP包使用了**加密协议**进行，基于VRRP技术，可以实现IP地址漂移，是一种容错协议，广泛应用于企业生产环境中。

### 3\. Nginx+Keepalive集群实战：

*   随着Nginx在国内的发展潮流，越来越多的互联网公司都在使用Nginx，Nginx高性能、稳定性成为IT人士青睐的HTTP和反向代理服务器。
*   Nginx负载均衡一般位于整个网站架构的**最前端**或者**中间层**，如果为最前端时**单台Nginx会存在单点故障**，也就是一台Nginx宕机，会影响用户对整个网站的访问。所以需要加入Nginx备份服务器，Nginx主服务器与备份服务器之间形成高可用，一旦发现Nginx主宕机，能快速将网站恢复至备份主机。Nginx+keepalived网络架构如图23-1所示：  
    ![](https://i-blog.csdnimg.cn/blog_migrate/8a29b55504d9f3fd1553f262783428f9.png)

### 3.1. 环境准备：

> nginx版本：nginx v1.18.0  
> keepalive版本：keepalive v1.3.5  
> Nginx-1:192.168.20.10(master)  
> Nginx-2:192.168.20.20(backup)

### 3.2. 安装软件包：

    #下载keepalive源码包：
    wget https://www.keepalived.org/software/keepalived-1.3.5.tar.gz
    #下载nginx源码包：
    wget http://nginx.org/download/nginx-1.18.0.tar.gz
    #解压：
    tar -xf keepalived-1.3.5.tar.gz -C /usr/src/
    tar -xf nginx-1.18.0.tar.gz -C /usr/src/
    #安装依赖包：
    yum -y install gcc-c++ pcre pcre-devel zlib zlib-devel openssl openssl-devel libnl libnl-devel libnfnetlink libnfnetlink-devel
    #创建nginx用户：
    useradd -s /sbin/nologin nginx -M
    #预编译nginx:
    cd /usr/src/nginx-1.18.0/
    ./configure --prefix=/usr/local/nginx --user=nginx --group=nginx --with-http_stub_status_module --with-http_ssl_module
    #编译&安装nginx
    make && make install
    #配置nginx环境变量：
    vim /etc/profile
    #后面添加如下内容：
    export PATH=$PATH:/usr/local/nginx/sbin
      source /etc/profile 
    #预编译keepalive:
    cd /usr/src/keepalived-1.3.5/
    ./configure --prefix=/usr/local/keepalived/ --with-kernel-dir=/usr/src/kernels/3.10.0-514.el7.x86_64/
    #编译&&安装keepalive：
    make && install
    #安装完成后，keepalived的默认配置文件地址和我们安装的地址不一样，所以复制过去就可以了
    cp /usr/src/keepalived-1.3.5/keepalived/etc/init.d/keepalived /etc/init.d/
    mkdir -p /etc/keepalived
    cp /usr/local/keepalived/etc/keepalived/keepalived.conf  /etc/keepalived/
    cp /usr/src/keepalived-1.3.5/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
    #配置环境变量
    vim /etc/profile
    #后面添加如下内容：
    export PATH=$PATH:/usr/local/keepalived/sbin
      source /etc/profile 


_注意：以上的配置master和backup都需要安装_

### 3.3. 配置keepalive:

* 配置Keepalived，两台服务器keepalived.conf内容都为如下，state均设置为backup，Backup服务器需要修改优先级为90。
````
    vim /etc/keepalived/keepalive.conf
    ! Configuration File for keepalived
       } 
       notification_email_from Alexandre.Cassen@firewall.loc
       smtp_server 127.0.0.1
       smtp_connect_timeout 30
       router_id LVS_DEVEL
    }  
    vrrp_script chk_nginx {
        script  "/data/sh/check_nginx.sh"
        interval 2 
        weight 2 
     }  
    vrrp_instance VI_1 {
        state BACKUP
        interface ens33
        virtual_router_id 51
        priority 90
        advert_int 5
        nopreempt
        authentication {
            auth_type PASS
            auth_pass 1111
        }   
        virtual_ipaddress {
            192.168.20.100
        }   
        track_script {
         chk_nginx 
        } 
    }

````


### 3.4. 创建脚本：

*   如上配置还需要建立check\_nginx脚本，用于检查本地Nginx是否存活，如果不存活，则kill keepalived实现切换。其中check\_nginx.sh脚本内容如下
```

    #创建存放脚本路径：
    mkdir -p /data/sh/
    #创建脚本：
    vim /data/sh/check_nginx.sh
    #!/bin/bash
    #auto  check  nginx  process
    #2020年9月15日17:16:29
    #by  author  XiaoYuEr
    killall  -0   nginx
    if  [[ $? -ne 0 ]];then
    service keepalived stop
    fi
    chmod +x /data/sh/check_nginx.sh
    ##killall这个命令没有需要安装psmisc-22.20-15.el7.x86_64即可
```


_注意：这个脚本需要两台电脑都需要创建_  
**启动keepalive时可能会遇到以下报错：**

    systemd: PID file /usr/local/keepalived/var/run/keepalived.pid not readable (yet?) after start.
    #这时需要修改keepalived.service文件把pid路径修改一下就可以了
    vim /lib/systemd/system/keepalived.service
    #修改为以下内容即可：
    PIDFile=/var/run/keepalived.pid


### 3.5. 测试：

*   在两台Nginx服务器分别新建index.html测试，一台nginx挂掉是否还可以访问：

    #在master中配置index.html
    [root@localhost ~]# echo "this is 192.168.20.10 page" > /usr/local/nginx/html/index.html
    #在backup机器中配置index.html
    echo "this is 192.168.20.20 page" > /usr/local/nginx/html/index.html


*   打开浏览器访问192.168.20.100：  
    ![](https://i-blog.csdnimg.cn/blog_migrate/ea729dc52ea593ef692c32ab9c26cffe.png)

*   把master主机nginx关掉再次访问看下：  
    ![](https://i-blog.csdnimg.cn/blog_migrate/cb9104e9a4a4cbf0414a419ed977902b.png)


### 3.6. Keepalived配置文件参数详解：
```
*   完整的keepalived的配置文件，其配置文件keepalived.conf可以包含三个文本块：**全局定义块**、**VRRP实例定义块**及**虚拟服务器定义块**。全局定义块和虚拟服务器定义块是必须的，如果在只有一个负载均衡器的场合，就不须VRRP实例定义块。

    [全局定义块]
    global_defs {
          notification_email {           			    --指定keepalived在发生切换时需要发送email到的对象，一行一个;
             wgkgood@gmail.com	                      
          }	
         notification_email_from  root@localhost	    --指定发件人
         smtp_server  127.0.0.1           			    --指定smtp服务器地址
         smtp_connect_timeout 3          	            --指定smtp连接超时时间
         router_id LVS_DEVEL             			    --运行keepalived机器的标识
    }	
    [监控Nginx进程]			
    vrrp_script	chk_nginx  {	
        script "/data/script/nginx.sh"      		    --监控服务脚本，脚本x执行权限；
        interval 2                    				    --检测时间间隔(执行脚本间隔)
        weight 2	                                    --权重
    }				
    [VRRP实例定义块]				
    vrrp_sync_group VG_1{                			    --监控多个网段的实例
            group {			 	
      VI_1                     			                --实例名1
      VI_2	
     }	
     notify_master /data/sh/nginx.sh          		    --指定当切换到master时，执行的脚本
     notify_backup /data/sh/nginx.sh          		    --指定当切换到backup时，执行的脚本
     notify   /data/sh/nginx.sh						    --发生任何切换，均执行的脚本
     smtp_alert                         			    --使用global_defs中提供的邮件地址和smtp服务器发送邮件通知；
    }		
    vrrp_instance VI_1 {		
        state BACKUP                    			    --设置主机状态，MASTER|BACKUP
        nopreempt                       			    --设置为不抢占
    interface eth0                   			        --对外提供服务的网络接口
    lvs_sync_daemon_inteface eth0                       --负载均衡器之间监控接口; 
        track_interface {               	 			--设置额外的监控，网卡出现问题都会切换；
         eth0	
         eth1	
        }	
        mcast_src_ip                    			    --发送组播包的地址，如果不设置默认使用绑定网卡的primary ip
        garp_master_delay              				    --在切换到master状态后，延迟进行gratuitous ARP请求
        virtual_router_id 50            			    --VRID标记 ,路由ID，可通过#tcpdump vrrp查看
        priority 90                    				    --优先级，优先级高者竞选为master
        advert_int 5                    			    --检查间隔，默认5秒
        preempt_delay                   			    --抢占延时，默认5分钟
        debug                           			    --debug日志级别
        authentication {                			    --设置认证
            auth_type PASS              			    --认证方式
            auth_pass 1111          				    --认证密码
        }
        track_script {                      		    --以脚本为监控chk_nginx；
            chk_nginx		
        }		
        virtual_ipaddress {             			    --设置vip地址
            192.168.111.188
        }
    }
    注意：使用了脚本监控Nginx或者MYSQL，不需要下面虚拟服务器设置块。
    [虚拟服务器定义块]
    virtual_server 192.168.111.188 3306 {
        delay_loop 6                   	               --健康检查时间间隔
        lb_algo rr                     	               --调度算法rr|wrr|lc|wlc|lblc|sh|dh
        lb_kind DR                     				   --负载均衡转发规则NAT|DR|TUN
        persistence_timeout  5        	     		   --会话保持时间
        protocol TCP                   				   --使用的协议
        real_server 192.168.1.12 3306 {	
                   weight 1            				   --默认为1,0为失效
                   notify_up   <string> | <quoted-string> --在检测到server up后执行脚本；
                   notify_down <string> | <quoted-string> --在检测到server down后执行脚本；
                   TCP_CHECK {
                   connect_timeout 3    		       --连接超时时间;
                   nb_get_retry  1     				   --重连次数;
                   delay_before_retry 1  			   --重连间隔时间;
                   connect_port 3306  				   --健康检查的端口;
                   }
           HTTP_GET {    
           url  {
              path /index.html          		       --检测url，可写多个
              digest  24326582a86bee478bac72d5af25089e --检测效验码
              --digest效验码获取方法：genhash -s IP -p 80 -u http://IP/index.html 
              status_code 200                          --检测返回http状态码
          }
    }
    }

  ```
    注意：使用了脚本监控Nginx或者MYSQL，不需要下面虚拟服务器设置块。
    [虚拟服务器定义块]
    virtual_server 192.168.111.188 3306 {
    delay_loop 6                   	               --健康检查时间间隔
    lb_algo rr                     	               --调度算法rr|wrr|lc|wlc|lblc|sh|dh
    lb_kind DR                     				   --负载均衡转发规则NAT|DR|TUN
    persistence_timeout  5        	     		   --会话保持时间
    protocol TCP                   				   --使用的协议
    real_server 192.168.1.12 3306 {
    weight 1            				   --默认为1,0为失效
    notify_up   <string> | <quoted-string> --在检测到server up后执行脚本；
    notify_down <string> | <quoted-string> --在检测到server down后执行脚本；
    TCP_CHECK {
    connect_timeout 3    		       --连接超时时间;
    nb_get_retry  1     				   --重连次数;
    delay_before_retry 1  			   --重连间隔时间;
    connect_port 3306  				   --健康检查的端口;
    }
    HTTP_GET {    
    url  {
    path /index.html          		       --检测url，可写多个
    digest  24326582a86bee478bac72d5af25089e --检测效验码
    --digest效验码获取方法：genhash -s IP -p 80 -u http://IP/index.html
    status_code 200                          --检测返回http状态码
    }
    }
    }


### 4\. 企业级Nginx+Keepalived双主架构实战

*   Nginx+keepalived主备模式，始终存在一台服务器处于**空闲状态**，如何更好的把两台服务器利用起来呢，可以借助Nginx+keepalived双主架构来实现，如图23-2所示，将架构改成**双主架构**，也即同时两台对外**两个VIP地址**，同时接收用户的请求。  
    ![](https://i-blog.csdnimg.cn/blog_migrate/9540fb6475287da8c73f195c12e72c97.png)

### 4.1. 环境准备：

> nginx版本：nginx v1.18.0  
> keepalive版本：keepalive v1.3.5  
> Nginx-1:192.168.20.10(master) （backup)  
> Nginx-2:192.168.20.20(backup) (master)

### 4.2. 配置如下：

*   以下内容是接着上面的配置来完成的。
````
    #以下配置是master端配置的，即192.168.20.10：
    ! Configuration File for keepalived

    global_defs {
    notification_email {
    acassen@firewall.loc
    }
    notification_email_from Alexandre.Cassen@firewall.loc
    smtp_server 127.0.0.1
    smtp_connect_timeout 30
    router_id LVS_DEVEL
    }
    vrrp_script chk_nginx {
    script  "/data/sh/check_nginx.sh"
    interval 2
    weight 2
    }
    #VIP1
    vrrp_instance VI_1 {
    state MASTER
    interface ens33
    lvs_sync_daemon_inteface ens33
    virtual_router_id 51
    priority 100
    advert_int 5
    nopreempt
    authentication {
    auth_type PASS
    auth_pass 1111
    }
    virtual_ipaddress {
    192.168.20.100
    }
    track_script {
    chk_nginx
    }
    }
    #VIP2
    vrrp_instance VI_2 {
    state BACKUP
    interface ens33
    lvs_sync_daemon_inteface ens33
    virtual_router_id 52
    priority  90
    advert_int 5
    nopreempt
    authentication {
    auth_type  PASS
    auth_pass  2222
    }
    virtual_ipaddress {
    192.168.20.200
    }
    track_script {
    chk_nginx
    }
    }
````

````
    #backup端配置即：192.168.20.20
    ! Configuration File for keepalived
    global_defs {
       notification_email {
         acassen@firewall.loc
       smtp_server 127.0.0.1
       smtp_connect_timeout 30
    
       notification_email {
         acassen@firewall.loc
       } 
       smtp_server 127.0.0.1
       smtp_connect_timeout 30
       router_id LVS_DEVEL
    }  
    vrrp_script chk_nginx {
        script  "/data/sh/check_nginx.sh"
        interval 2 
        weight 2 
     }  
    #VIP1
    vrrp_instance VI_1 {
        state BACKUP
        interface ens33
        lvs_sync_daemon_inteface ens33
        virtual_router_id 51
        priority 90
        advert_int 5
        nopreempt
        authentication {
            auth_pass 1111
        }   
        virtual_ipaddress {
            192.168.20.100
        }   
        track_script {
         chk_nginx 
        } 
    }   
    #VIP2
    vrrp_instance VI_2 {
         state MASTER  
         interface ens33
         lvs_sync_daemon_inteface ens33
         virtual_router_id  52 
         priority  100
         advert_int 5 
         nopreempt  
         authentication {
             auth_type  PASS
             auth_pass  2222
         }   
         virtual_ipaddress {
             192.168.20.200 
         }   
         track_script {
         chk_nginx 
        } 
     } 
````





*   Nginx+keepalived双主企业架构，在**日常维护及管理过程**中需要如下几个方面：

> 1.  Keepalived主配置文件必须设置不同的VRRP名称，同时优先级和VIP设置也各不相同；
> 2.  Nginx网站总访问量为两台Nginx服务器之和，可以写脚本自动统计访问量；
> 3.  两台Nginx为Master，存在两个VIP地址，用户从外网访问VIP，需配置域名映射到两个VIP上方可。
> 4.  通过外网DNS映射不同VIP的方法也称为DNS负载均衡模式； 可以通过Zabbix实时监控VIP访问状态是否正常。



文章知识点与官方知识档案匹配，可进一步学习相关知识

[CS入门技能树](https://edu.csdn.net/skill/gml/gml-1c31834f07b04bcc9c5dff5baaa6680c?utm_source=csdn_ai_skill_tree_blog)[Linux入门](https://edu.csdn.net/skill/gml/gml-1c31834f07b04bcc9c5dff5baaa6680c?utm_source=csdn_ai_skill_tree_blog)[初识Linux](https://edu.csdn.net/skill/gml/gml-1c31834f07b04bcc9c5dff5baaa6680c?utm_source=csdn_ai_skill_tree_blog)41758 人正在系统学习中
