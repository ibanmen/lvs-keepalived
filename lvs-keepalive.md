集群高可用之lvs+keepalive
-------------------------

#### keepalive简介：

-	负载均衡架构依赖于知名的IPVS内核模块，keepalive由一组检查器根据服务器的健康情况动态维护和管理服务器池。keepalive通过VRRP协议实现高可用架构。VRRP是路由灾备的实现基础。

-	LVS核心是调度器，所有的数据请求需要经过调度器进行调度转发。万一调度器发生故障，整个集群系统全部崩溃，所以需要keepalive实现集群系统的高可用。

-	部署两台或多台lvs调度器，仅有一台调度器做主服务器，其他为备用。当主发生故障后，keepalive可以自动将备用调度器作为主，实现整个集群系统的高负载，高可用

#### VRRP协议介绍：

-	VRRP协议是为消除在静态缺省路由环境下的缺省路由器单点故障引起的网络失效而设计的主备模式的协议，使得在发生故障而进行设备功能切换时可以不影响内外数据通信，不需要再修改内部网络的网络参数。VRRP协议需要具有IP地址备份，优先路由选择，减少不必要的路由器间通信等功能。

-	VRRP协议将两台或多台路由器设备虚拟成一个设备，对外提供虚拟路由器IP(一个或多个)，而在路由器组内部，如果实际拥有这个对外IP的路由器如果工作正常的话就是MASTER，或者是通过算法选举产生，MASTER实现针对虚拟路由器IP的各种网络功能，如ARP请求，ICMP，以及数据的转发等；其他设备不拥有该IP，状态是BACKUP，除了接收MASTER的VRRP状态通告信息外，不执行对外的网络功能。当主机失效时，BACKUP将接管原先MASTER的网络功能。

-	配置VRRP协议时需要配置每个路由器的虚拟路由器ID(VRID)和优先权值，使用VRID将路由器进行分组，具有相同VRID值的路由器为同一个组，VRID是一个0～255的正整数；同一组中的路由器通过使用优先权值来选举MASTER，优先权大者为MASTER，优先权也是一个0～255的正整数。

-	有一种特殊情况，将虚拟路由IP地址设置为多台路由设备中某台设备的真实ip地址，该路由设备将永远处于主设备状态。

注：本例用centos7服务器

#### keepalived 搭建

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
