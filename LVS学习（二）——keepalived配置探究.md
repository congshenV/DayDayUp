LVS学习（二）——keepalived配置探究
===========================



在部署keepalived和LVS的时候，没有了解部署原理及配置原理，故而带来了一些问题。
所以今天从线上变更keepalived配置文件为目的，做了一些探究，解决以下两个问题：
问题1、使用group模式的配置来部署单实例，造成group里面的notify命令不起作用，主从切换后连接同步程序的主从状态无法转换；
问题2、syncid与现有的syncid重复，可能带来不同lvs组之间的同步连接数据错乱；
故而，需要在线上手动去变更配置，解决以上两个问题带来的隐患。

****
	
|Author|葱shen|
|---|---|
|E-mail|yycmessi@163.com



****
## 今日内容预览
* [keepalived配置简介](#keepalived配置简介)
* [线上keepalived变更流程](#线上keepalived变更流程)
* [关于priority探究](#关于priority探究)




### keepalived配置简介
-----------
keepalived有三类配置区域(姑且就叫区域吧)，注意不是三种配置文件，是一个配置
文件里面三种不同类别的配置区域：
>全局配置(Global Configuration)
VRRPD配置
LVS配置

一．全局配置
全局配置又包括两个子配置：
全局定义(global definition)
静态路由配置(static ipaddress/routes)
1．全局定义(global definition)配置范例

    global_defs
    {
        notification_email
        {
            admin@example.com
        }
        notification_email_from admin@example.com
        smtp_server 127.0.0.1
        stmp_connect_timeout 30
        router_id node1
    }

全局配置解析
global_defs全局配置标识，表面这个区域{}是全局配置

    notification_email
    {
    admin@example.com
    admin@ywlm.net
    }

表示keepalived在发生诸如切换操作时需要发送email通知，以及email发送给哪些邮
件地址，邮件地址可以多个，每行一个。
notification_email_from admin@example.com：表示发送通知邮件时邮件源地址是谁
smtp_server 127.0.0.1：表示发送email时使用的smtp服务器地址，这里可以用本地
的sendmail来实现
smtp_connect_timeout 30：连接smtp连接超时时间
router_id node1：机器标识


二．VRRPD配置
VRRPD配置包括三个类：
VRRP同步组(synchroization group)
VRRP实例(VRRP Instance)
VRRP脚本

1、VRRP同步组(synchroization group)配置范例

    vrrp_sync_group VG_1 {
    group {
    http
    mysql
    }
    notify_master /path/to/to_master.sh
    notify_backup /path_to/to_backup.sh
    notify_fault "/path/fault.sh VG_1"
    notify /path/to/notify.sh
    smtp_alert
    }

其中：

    group {
    http
    mysql
    }

http和mysql是实例名和下面的实例名一致

    notify_master /path/to/to_master.sh：表示当切换到master状态时，要执行的脚本
    notify_backup /path_to/to_backup.sh：表示当切换到backup状态时，要执行的脚本
    notify_fault "/path/fault.sh VG_1"
    notify /path/to/notify.sh：
    smtp alter表示切换时给global defs中定义的邮件地址发送邮件通知

2，VRRP实例(instance)配置范例

    vrrp_instance http {
    state MASTER
    interface eth0
    dont_track_primary
    track_interface {
    eth0
    eth1
    }
    mcast_src_ip <IPADDR>
    garp_master_delay 10
    virtual_router_id 51
    priority 100
    advert_int 1
    authentication {
    auth_type PASS
    autp_pass 1234
    }
    virtual_ipaddress {
    #<IPADDR>/<MASK> brd <IPADDR> dev <STRING> scope <SCOPT> label <LABEL>
    192.168.200.17/24 dev eth1
    192.168.200.18/24 dev eth2 label eth2:1
    }
    virtual_routes {
    # src <IPADDR> [to] <IPADDR>/<MASK> via|gw <IPADDR> dev <STRING> scope
    <SCOPE> tab
    src 192.168.100.1 to 192.168.109.0/24 via 192.168.200.254 dev eth1
    192.168.110.0/24 via 192.168.200.254 dev eth1
    192.168.111.0/24 dev eth2
    192.168.112.0/24 via 192.168.100.254
    }
    nopreempt
    
    preemtp_delay 300
    debug
    }

>state：state指定instance(Initial)的初始状态，就是说在配置好后，这台服务器的
初始状态就是这里指定的，但这里指定的不算，还是得要通过竞选通过优先级来确
定，如果这里设置为master，但如若他的优先级不及另外一台，那么这台在发送通告
时，会发送自己的优先级，另外一台发现优先级不如自己的高，那么他会就回抢占为
master
interface：实例绑定的网卡，因为在配置虚拟IP的时候必须是在已有的网卡上添加的
dont track primary：忽略VRRP的interface错误
track interface：跟踪接口，设置额外的监控，里面任意一块网卡出现问题，都会进
入故障(FAULT)状态，例如，用nginx做均衡器的时候，内网必须正常工作，如果内网
出问题了，这个均衡器也就无法运作了，所以必须对内外网同时做健康检查
mcast src ip：发送多播数据包时的源IP地址，这里注意了，这里实际上就是在那个
地址上发送VRRP通告，这个非常重要，一定要选择稳定的网卡端口来发送，这里相当
于heartbeat的心跳端口，如果没有设置那么就用默认的绑定的网卡的IP，也就是
interface指定的IP地址
garp master delay：在切换到master状态后，延迟进行免费的ARP(gratuitous ARP)
请求
virtual router id：这里设置VRID，这里非常重要，相同的VRID为一个组，他将决定
虚拟MAC地址
priority 100：设置本节点的优先级，优先级高的为master
advert int：检查间隔(组播信息发送间隔)，默认为1秒
virtual ipaddress：这里设置的就是VIP，也就是虚拟IP地址，他随着state的变化而
增加删除，当state为master的时候就添加，当state为backup的时候删除，这里主要
是由优先级来决定的，和state设置的值没有多大关系，这里可以设置多个IP地址
virtual routes：原理和virtual ipaddress一样，只不过这里是增加和删除路由
lvs sync daemon interface：lvs syncd绑定的网卡
authentication：这里设置认证
auth type：认证方式，可以是PASS或AH两种认证方式
auth pass：认证密码
nopreempt：设置不抢占，这里只能设置在state为backup的节点上，而且这个节点的
优先级必须别另外的高
preempt delay：抢占延迟
debug：debug级别
notify master：和sync group这里设置的含义一样，可以单独设置，例如不同的实例
通知不同的管理人员，http实例发给网站管理员，mysql的就发邮件给DBA


3．VRRP脚本

    vrrp_script check_running {
    script "/usr/local/bin/check_running"
    interval 10
    weight 10
    }
    vrrp_instance http {
    state BACKUP
    smtp_alert
    interface eth0
    virtual_router_id 101
    priority 90
    advert_int 3
    authentication {
    auth_type PASS
    auth_pass whatever
    }
    virtual_ipaddress {
    1.1.1.1
    }
    track_script {
    check_running weight 20
    } }

首先在vrrp_script区域定义脚本名字和脚本执行的间隔和脚本执行的优先级变更
vrrp_script check_running {
script "/usr/local/bin/check_running"
interval 10 #脚本执行间隔
weight 10 #脚本结果导致的优先级变更：10表示优先级+10；-10则表示优先
级-10
}
然后在实例(vrrp_instance)里面引用，有点类似脚本里面的函数引用一样：先定义，
后引用函数名
track_script {
check_running weight 20
}
注意：VRRP脚本(vrrp_script)和VRRP实例(vrrp_instance)属于同一个级别

### 线上keepalived变更流程
-----------
**重要说明: 由于是线上操作，务必谨慎，多检查**

1、检查连接同步情况，及主从角色状态：

    ipvsadm -ln
    ipvsadm -l --daemon
    ip a

2、修改配置文件keepalived.conf，将group模式改成instance模式，并更改syncid：
    最后完整的配置为：（从库在下面标注的地方修改即可）
    ! Configuration File for keepalived
    
    global_defs {
         router_id Codis_LB
    }
    
    vrrp_instance codis_6379 {
        state MASTER   ！从库为 SALVE
        interface eth0
        virtual_router_id 79
        priority 100    ！从库为 101
        advert_int 1
        nopreempt
        unicast_peer {
            127.0.0.2    ！从库为 127.0.0.1
        }
        authentication {
            auth_type PASS
            auth_pass codis79
        }
        virtual_ipaddress {
            127.0.0.100
        }
        notify_master "notify.sh MASTER eth0 11 > /dev/null 2>&1"
        notify_backup "notify.sh BACKUP eth0 11 > /dev/null 2>&1"
        notify_fault  "notify.sh FAULT eth0 11 > /dev/null 2>&1"
    }
    
    virtual_server 127.0.0.100 6379 {
        delay_loop 6
        lb_algo wrr
        lb_kind DR
        protocol TCP
        include realserver_6379.conf
    }
    
3、先systemctl reload keepalived从127.0.0.2，后主127.0.0.1，检查服务是否正常启动，同步是否正常，查看message日志，

    ipvsadm -ln
    ipvsadm -l --daemon
    vim /var/log/messages
    
4、检查服务是否正常启动，同步是否正常,查看message日志,查看连接、流量、是否正常；
   

        ipvsadm -ln
        ipvsadm -l --daemon
        vim /var/log/messages
        ipvsadm -lnc |grep 6379|awk '{print $4}'|awk -F ":" '{print $1}'|sort |uniq

5、连接同步syncid变更，先从后主：

    从执行：
    ipvsadm --stop-daemon backup
    ipvsadm --start-daemon backup --mcast-interface eth0 --syncid 11

    主执行：
    ipvsadm --stop-daemon master
    ipvsadm --start-daemon master --mcast-interface eth0 --syncid 11
    
    检查同步是否正常：
    ipvsadm -ln
    ipvsadm -l --daemon
    
    检查流量：watch -n 1 ipvsadm -ln --rate
    检查启动情况：tail -f /var/log/messages
    
    
### 关于priority探究
-----------
关于priority配置backup比master大的原因：
此参数关系到选举，什么时候会触发选举？
避免backup升为主后，旧master启动之后抢占，所以初始master的权重应该设置低一些，保证再次进行切换
    
通常如果master服务死掉后backup会变成master，但是当master服务又好了的时候 master此时会抢占VIP，这样就会发生两次切换对业务繁忙的网站来说是不好的。所以我们要在配置文件加入 nopreempt 非抢占，但是这个参数只能用于state 为backup，故我们在用HA的时候最好master 和backup的state都设置成backup 让其通过priority来竞争。
    
下面演示三种情形下的主从切换：
>route A, priority=100, MASTER, nopreempt
route B, priority=101, BACKUP, nopreempt
1. 先启动A, A进入MASTER状态
2. 再启动B, B进入BACKUP状态, 进入BACKUP后, 不主动竞选MASTER(因为设置了nopreempt), 所以A还是MASTER.
3. 如果A异常, 则B变成MASTER
4. 如果A恢复, A申请竞选MASTER, 但是优先级低于B, 所以进入BACKUP, 进入BACKUP后, 就不会主动竞选MASTER了.
5. 如果B异常，则A变成MASTER
6. 如果B恢复，B先进入BACKUP状态, 进入BACKUP后, 由于设置了nopreempt，不会竞选MASTER, 所以A还是MASTER


----------


>route A, priority=101, MASTER, nopreempt
route B, priority=100, BACKUP, nopreempt
1. 先启动A, A进入MASTER状态
2. 再启动B, B进入BACKUP状态, 进入BACKUP后, 不主动竞选MASTER(因为设置了nopreempt), 所以A还是MASTER.
3. 如果A异常, 则B变成MASTER
4. 如果A恢复, A申请竞选MASTER, 优先级高于B, 所以竞选成功，发生主从切换，A进入MASTER状态，B进入BACKUP状态, 


----------


>route A, priority=101, BACKUP, nopreempt
route B, priority=100, BACKUP, nopreempt
1. 先启动A, A初始化BACKUP状态，由于只有A，所以A进入MASTER状态；
2. 再启动B, B进入BACKUP状态, 进入BACKUP后, 不主动竞选MASTER(因为设置了nopreempt), 所以A还是MASTER.
3. 如果A异常, 则B变成MASTER
4. 如果A恢复, A先进入BACKUP状态, 进入BACKUP后, 由于设置了nopreempt，不会竞选MASTER, 所以B还是MASTER；
5. 如果B异常，则A变成MASTER
6. 如果B恢复，B先进入BACKUP状态, 进入BACKUP后, 由于设置了nopreempt，不会竞选MASTER, 所以A还是MASTER