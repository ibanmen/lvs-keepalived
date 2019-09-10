## 前言
>Keepalived使用的vrrp协议方式，虚拟路由冗余协议 (Virtual Router Redundancy Protocol，简称VRRP)；Heartbeat或Corosync是基于主机或网络服务的高可用方式；简单的说就是，Keepalived的目的是模拟路由器的高可用，Heartbeat或Corosync的目的是实现Service的高可用。所以一般Keepalived是实现前端高可用，常用的前端高可用的组合有，就是我们常见的LVS+Keepalived、Nginx+Keepalived、HAproxy+Keepalived。而Heartbeat或Corosync是实现服务的高可用，常见的组合有Heartbeat v3(Corosync)+Pacemaker+NFS+Httpd 实现Web服务器的高可用、Heartbeat v3(Corosync)+Pacemaker+NFS+MySQL 实现MySQL服务器的高可用。总结一下，Keepalived中实现轻量级的高可用，一般用于前端高可用，且不需要共享存储，一般常用于两个节点的高可用。而Heartbeat(或Corosync)一般用于服务的高可用，且需要共享存储，一般用于多节点的高可用。这个问题我们说明白了，又有博友会问了，那heartbaet与corosync我们又应该选择哪个好啊，我想说我们一般用corosync，因为corosync的运行机制更优于heartbeat，就连从heartbeat分离出来的pacemaker都说在以后的开发当中更倾向于corosync，所以现在corosync+pacemaker是最佳组合。但说实话我对于软件没有任何倾向性，所以我把所有的集群软件都和大家说了一下，我认为不管什么软件，只要它能存活下来都有它的特点和应用领域，只有把特定的软件放在特定的位置才能发挥最大的作用，那首先我们得对这个软件有所有了解。学习一种软件的最好方法，就是去查官方文档。好了说了那么多希望大家有所收获，下面我们来说一说keepalived。

## 目录

- [前言](#前言)
- [keepalive简介](#keepalive简介)
- [keepalived 搭建](#keepalived 搭建)


## keepalive简介：
### 1.Keepalived 定义
>Keepalived 是一个基于VRRP协议来实现的LVS服务高可用方案，可以利用其来避免单点故障。一个LVS服务会有2台服务器运行Keepalived，一台为主服务器（MASTER），一台为备份服务器（BACKUP），但是对外表现为一个虚拟IP，主服务器会发送特定的消息给备份服务器，当备份服务器收不到这个消息的时候，即主服务器宕机的时候， 备份服务器就会接管虚拟IP，继续提供服务，从而保证了高可用性。Keepalived是VRRP的完美实现，因此在介绍keepalived之前，先介绍一下VRRP的原理。
-	负载均衡架构依赖于知名的IPVS内核模块，keepalive由一组检查器根据服务器的健康情况动态维护和管理服务器池。keepalive通过VRRP协议实现高可用架构。VRRP是路由灾备的实现基础。
-	LVS核心是调度器，所有的数据请求需要经过调度器进行调度转发。万一调度器发生故障，整个集群系统全部崩溃，所以需要keepalive实现集群系统的高可用。
-	部署两台或多台lvs调度器，仅有一台调度器做主服务器，其他为备用。当主发生故障后，keepalive可以自动将备用调度器作为主，实现整个集群系统的高负载，高可用
### 2.VRRP 协议简介
在现实的网络环境中，两台需要通信的主机大多数情况下并没有直接的物理连接。对于这样的情况，它们之间路由怎样选择？主机如何选定到达目的主机的下一跳路由，这个问题通常的解决方法有二种：
<br>在主机上使用动态路由协议(RIP、OSPF等)
<br>在主机上配置静态路由
<br>很明显，在主机上配置动态路由是非常不切实际的，因为管理、维护成本以及是否支持等诸多问题。配置静态路由就变得十分流行，但路由器(或者说默认网关default gateway)却经常成为单点故障。VRRP的目的就是为了解决静态路由单点故障问题，VRRP通过一竞选(election)协议来动态的将路由任务交给LAN中虚拟路由器中的某台VRRP路由器。
### 3.VRRP 工作机制
>在一个VRRP虚拟路由器中，有多台物理的VRRP路由器，但是这多台的物理的机器并不能同时工作，而是由一台称为MASTER的负责路由工作，其它的都是BACKUP，MASTER并非一成不变，VRRP让每个VRRP路由器参与竞选，最终获胜的就是MASTER。MASTER拥有一些特权，比如，拥有虚拟路由器的IP地址，我们的主机就是用这个IP地址作为静态路由的。拥有特权的MASTER要负责转发发送给网关地址的包和响应ARP请求。
>VRRP通过竞选协议来实现虚拟路由器的功能，所有的协议报文都是通过IP多播(multicast)包(多播地址224.0.0.18)形式发送的。虚拟路由器由VRID(范围0-255)和一组IP地址组成，对外表现为一个周知的MAC地址。所以，在一个虚拟路由 器中，不管谁是MASTER，对外都是相同的MAC和IP(称之为VIP)。客户端主机并不需要因为MASTER的改变而修改自己的路由配置，对客户端来说，这种主从的切换是透明的。
>在一个虚拟路由器中，只有作为MASTER的VRRP路由器会一直发送VRRP通告信息(VRRPAdvertisement message)，BACKUP不会抢占MASTER，除非它的优先级(priority)更高。当MASTER不可用时(BACKUP收不到通告信息)， 多台BACKUP中优先级最高的这台会被抢占为MASTER。这种抢占是非常快速的(<1s)，以保证服务的连续性。由于安全性考虑，VRRP包使用了加密协议进行加密。
### 4.VRRP 工作流程
#### (1).初始化：    
<br>路由器启动时，如果路由器的优先级是255(最高优先级，路由器拥有路由器地址)，要发送VRRP通告信息，并发送广播ARP信息通告路由器IP地址对应的MAC地址为路由虚拟MAC，设置通告信息定时器准备定时发送VRRP通告信息，转为MASTER状态；否则进入BACKUP状态，设置定时器检查定时检查是否收到MASTER的通告信息。
#### (2).Master
- 设置定时通告定时器；
- 用VRRP虚拟MAC地址响应路由器IP地址的ARP请求；
- 转发目的MAC是VRRP虚拟MAC的数如果是虚拟路由器IP的拥有者，将接受目的地址是虚拟路由器IP的数据当收到shutdown的事件时删除定时通告定时器，发送优先权<级为0的如果定时通告定时器超时时，发送VRRP通告信息；
- 收到VRRP通告信息时，如果优先权为0，发送VRRP通告信息；否则判断数据的优先级是否高于本机，或相等而且实际IP地址大于本地实际IP，设置定时通告定时器，复位主机超时定时器，转BACKUP状态；否则的话，丢弃该通告包；
#### (3).Backup
- 设置主机超时定时器；
- 不能响应针对虚拟路由器IP的ARP请求信息；
- 丢弃所有目的MAC地址是虚拟路由器MAC地址的数据包；
- 不接受目的是虚拟路由器IP的所有数据包；
- 当收到shutdown的事件时删除主机超时定时器，转初始化状态；
- 主机超时定时器超时的时候，发送VRRP通告信息，广播ARP地址信息，转MASTER状态；
- 收到VRRP通告信息时，如果优先权为0，表示进入MASTER选举；否则判断数据的优先级是否高于本机，如果高的话承认MASTER有效，复位主机超时定时器；否则的话，丢弃该通告包；
### 5.ARP查询处理
> 当内部主机通过ARP查询虚拟路由器IP地址对应的MAC地址时，MASTER路由器回复的MAC地址为虚拟的VRRP的MAC地址，而不是实际网卡的 MAC地址，这样在路由器切换时让内网机器觉察不到；而在路由器重新启动时，不能主动发送本机网卡的实际MAC地址。如果虚拟路由器开启的ARP代理 (proxy_arp)功能，代理的ARP回应也回应VRRP虚拟MAC地址；好了VRRP的简单讲解就到这里，我们下来讲解一下Keepalived的案例。

## keepalived 搭建

![](https://ws1.sinaimg.cn/large/006jQWURgy1g61am0adejj30si0kuqat.jpg)

配置图如上：<br> 配置如下：<br> web1，web2 的操作

##### 1、关闭防火墙并禁止开机自启

```
$ systemctl stop firewalld.service
$ systemctl disable firewalld
或者以下方案
firewall-cmd --direct --permanent --add-rule ipv4 filter INPUT 0 --in-interface ens33 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
firewall-cmd --direct --permanent --add-rule ipv4 filter OUTPUT 0 --out-interface ens33 --destination 224.0.0.18 --protocol vrrp -j ACCEPT
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --reload
firewall-cmd --list-ports
```

##### 2、关闭selinux

```
$ sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
重启 reboot
```

##### 3、关闭NetworkManager

```
$ systemctl stop NetworkManager.service
$ systemctl disable NetworkManager
```

##### 4、修改hostname

```
hostnamectl set-hostname master
hostnamectl set-hostname backup
hostnamectl set-hostname node1
hostnamectl set-hostname node2
```

###### 4、编写lvs_rs启动脚本（web1，web2 两台都要操作）

```
firewall-cmd --add-port=80/tcp --permanent
firewall-cmd --reload
```

$ vim /etc/init.d/realserver

```
#!/bin/bash
#
# lvs      Start lvs
#
# chkconfig: 2345 08 92
# description:  Starts, stops and saves lvs
#
VIP=192.168.50.200
. /etc/rc.d/init.d/functions

case "$1" in
start)
  /sbin/ifconfig lo down
  /sbin/ifconfig lo up
  echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
  echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
  echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
  echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
  /sbin/sysctl -p >/dev/null 2>&1
  /sbin/ifconfig lo:0 $VIP netmask 255.255.255.255 up  
  /sbin/route add -host $VIP dev lo:0
  echo "LVS-DR real server starts successfully.\n"
  ;;
stop)
  /sbin/ifconfig lo:0 down
  /sbin/route del $VIP >/dev/null 2>&1
  echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
  echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
  echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
  echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
echo "LVS-DR real server stopped.\n"
  ;;
status)
  isLoOn=`/sbin/ifconfig lo:0 | grep "$VIP"`
  isRoOn=`/bin/netstat -rn | grep "$VIP"`
  if [ "$isLoON" == "" -a "$isRoOn" == "" ]; then
      echo "LVS-DR real server has run yet."
  else
      echo "LVS-DR real server is running."
  fi
  exit 3
  ;;
*)
  echo "Usage: $0 {start|stop|status}"
  exit 1
esac
exit 0
```

将lvs脚本加入开机自启动

```
  $ chmod +x /etc/init.d/realserver
  $ echo "/etc/init.d/realserver start" >> /etc/rc.d/rc.local
  $ chmod +x /etc/rc.d/rc.local
```

启动LVS脚本(注意：如果这两台realserver机器重启了，一定要确保service realserver start 启动了，即lo:0本地回环上绑定了vip地址，否则lvs转发失败！)<br> service realserver start

##### 安装web服务

```
  $ rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
  $ yum install -y nginx
  $ systemctl start nginx.service
  $ systemctl enable nginx.service
  修改index为IP地址显示
  node1:
  echo "192.168.1.30" > /usr/local/nginx/html/index.html
  node2:
  echo "192.168.1.40" > /usr/local/nginx/conf/html/index.html
```

##### 5、lvs安装ipvsadm keepalived（主备两台都要操作）

```
  $ yum install -y curl gcc openssl-devel libnl3-devel net-snmp-devel ipvsadm
  $ curl --progress https://www.keepalived.org/software/keepalived-2.0.18.tar.gz | tar xz
  $ cd keepalived-2.0.18
  $ ./configure --prefix=/usr/local/keepalived
  $ make && make install
  $ ln -s /usr/local/keepalived/sbin/keepalived /usr/sbin
  $ mkdir /etc/keepalived/
  $ ln -s /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
  $ ln -s /usr/local/etc/rc.d/init.d/keepalived /etc/init.d/

  查看lvs集群
  $ ipvsadm -L -n
  IP Virtual Server version 2.0.18 (size=4096)
  Prot LocalAddress:Port Scheduler Flags
    -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
```

##### 6、配置keepalived

```
  主备打开ip_forward转发功能
  $ echo "1" > /proc/sys/net/ipv4/ip_forward
  主的keepalive.conf配置
```

vim /etc/keepalived/keepalived.conf

```
! Configuration File for keepalived

global_defs {
   router_id LVS_Master
}

vrrp_instance VI_1 {
    state MASTER               #指定instance初始状态，实际根据优先级决定.backup节点不一样
    interface ens33             #虚拟IP所在网
    virtual_router_id 66       #VRID，相同VRID为一个组，决定多播MAC地址
    priority 100               #优先级，另一台改为80.backup节点不一样
    advert_int 1               #检查间隔
    authentication {
        auth_type PASS         #认证方式，可以是pass或ha
        auth_pass 1111         #认证密码
    }
    virtual_ipaddress {
        192.168.1.100         #VIP
    }
}

virtual_server 192.168.1.100 80 {
    delay_loop 6               #服务轮询的时间间隔
    lb_algo wrr                #加权轮询调度，LVS调度算法 rr|wrr|lc|wlc|lblc|sh|sh
    lb_kind DR                 #LVS集群模式 NAT|DR|TUN，其中DR模式要求负载均衡器网卡必须有一块与物理网卡在同一个网段
    persistence_timeout 50     #会话保持时间
    protocol TCP              #健康检查协议

    real_server 192.168.1.30 80 {
        weight 3  ##权重
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
    }
    real_server 192.168.1.40 80 {
        weight 3
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
    }
}
```

$ systemctl start keepalived <br> $ systemctl enable keepalived <br> $ ip addr show <br> $ ipvsadm -Ln <br> 备用上的keepalived.conf配置 <br> $ vim /etc/keepalived/keepalived.conf <br> ! Configuration File for keepalived

```
global_defs {
   router_id LVS_Backup
}

vrrp_instance VI_1 {
    state BACKUP               #指定instance初始状态，实际根据优先级决定.backup节点不一样
    interface ens33             #虚拟IP所在网
    virtual_router_id 66       #VRID，相同VRID为一个组，决定多播MAC地址
    priority 80               #优先级，另一台改为90.master节点不一样
    advert_int 1               #检查间隔
    authentication {
        auth_type PASS         #认证方式，可以是pass或ha
        auth_pass 1111         #认证密码
    }
    virtual_ipaddress {
        192.168.1.100         #VIP
    }
}

virtual_server 192.168.1.100 80 {
    delay_loop 6               #服务轮询的时间间隔
    lb_algo wrr                #加权轮询调度，LVS调度算法 rr|wrr|lc|wlc|lblc|sh|sh
    lb_kind DR                 #LVS集群模式 NAT|DR|TUN，其中DR模式要求负载均衡器网卡必须有一块与物理网卡在同一个网段
    persistence_timeout 50     #会话保持时间
    protocol TCP              #健康检查协议

    real_server 192.168.1.30 80 {
        weight 3  ##权重
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
    }
    real_server 192.168.1.40 80 {
        weight 3
        TCP_CHECK {
            connect_timeout 3
            nb_get_retry 3
            delay_before_retry 3
            connect_port 80
        }
    }
}
```

$ systemctl start keepalived <br> $ systemctl enable keepalived <br> $ ip addr show <br> $ ipvsadm -Ln <br>

> 测试： 1）测试LVS功能（上面Keepalived的lvs配置中，自带了健康检查，当后端服务器的故障出现故障后会自动从lvs集群中踢出，当故障恢复后，再自动加入到集群中）<br> 测试关闭一台Real Server,比如Real_Server2<br> 过一会儿再次查看当前LVS集群，发现Real_Server2已经被踢出当前LVS集群了<br> 最后重启Real_Server2的80端口，发现LVS集群里又再次将其添加进来了<br> 2）测试Keepalived心跳测试的高可用<br> 默认情况下，VIP资源是在LVS_Keepalived_Master上<br> 然后关闭LVS_Keepalived_Master的keepalived，发现VIP就会转移到LVS_Keepalived_Backup上。<br> 查看系统日志，能查看到LVS_Keepalived_Master的VIP的移动信息<br>[root@LVS_Keepalived_Master ~]# tail -f /var/log/messages<br> 接着再重新启动LVS_Keepalived_Master的keepalived，发现VIP又转移回来了<br> 查看系统日志，能查看到LVS_Keepalived_Master的VIP转移回来的信息<br>[root@LVS_Keepalived_Master ~]# tail -f /var/log/messages
