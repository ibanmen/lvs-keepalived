### LVS DR模式搭建

准备工作：三台机器

分发器，也叫调度器（简写为dir）：192.168.248.128<br> rs1 ：192.168.248.129<br> rs2 : 192.168.248.130<br> vip : 192.168.248.200

配置与物理机处于同一vlan，防火墙关闭，selinux设置为disabled NetworkManager 关闭

##### 1.dir上编辑脚本文件**/usr/local/sbin/lvs_dr.sh**，文件内容如下：

```
#! /bin/bash
echo 1 > /proc/sys/net/ipv4/ip_forward #打开端口转发
ipv=/usr/sbin/ipvsadm
vip=192.168.248.200
rs1=192.168.248.132
rs2=192.168.248.133
#注意这里的网卡名字
#绑定vip
ifdown ens33
ifup ens33
ifconfig ens33:2 $vip broadcast $vip netmask 255.255.255.255 up
route add -host $vip dev ens33:2
$ipv -C
$ipv -A -t $vip:80 -s rr
$ipv -a -t $vip:80 -r $rs1:80 -g -w 1
$ipv -a -t $vip:80 -r $rs2:80 -g -w 1
```

##### 2.执行脚本

[root@yolks-001 ~]# sh /usr/local/sbin/lvs_dr.sh > 成功断开设备 'ens33'。 连接已成功激活（D-Bus 活动路径：/org/freedesktop/NetworkManager/ActiveConnection/5）

##### 3.rs机器也需要编辑配置文件，

添加脚本文件**/usr/local/sbin/lvs_rs.sh**，内容如下： #/bin/bash vip=192.168.248.200 #把vip绑定在lo上，是为了实现rs直接把结果返回给客户端 ifdown lo ifup lo ifconfig lo:0 $vip broadcast $vip netmask 255.255.255.255 up route add -host $vip lo:0 #以下操作为更改arp内核参数，目的是为了让rs顺利发送mac地址给客户端 #参考文档www.cnblogs.com/lgfeng/archive/2012/10/16/2726308.html echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce

##### 5、执行脚本

```
$ chmod +x /etc/init.d/lvsrs
$ echo "/etc/init.d/lvsrs.sh start" >> /etc/rc.d/rc.local
chmod +x /etc/rc.d/rc.local
```

##### 6.安装nginx

```
rpm -Uvh http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm
yum install -y nginx
systemctl start nginx.service
systemctl enable nginx.service
node1:
echo "192.168.248.129" > /usr/local/nginx/html/index.html
node2:
echo "192.168.248.130" > /usr/local/nginx/conf/html/index.html
```

##### 测试

访问网页192.168.248.128 查看网页显示内容
