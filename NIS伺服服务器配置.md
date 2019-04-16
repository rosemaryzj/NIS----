### NIS功能概述

```shell
NIS服务的应用结构分为NIS服务器和NIS客户机两种角色，NIS服务器集中维护用户的帐号信息（数据库）供NIS客户机进行查询，用户登录任何一台NIS客户机都会从NIS服务器进行登录认证，可实现用户帐号的集中管理
```

### NIS运行流程

```shell
Nis Server（Master/Slave）

1.Nis Master先将帐号密码相关文件制作成数据库文件；

2.Nis Master可以主动告诉Nis Slave来更新；

3.Nis Slave亦可以主动前往Nis Master取得更新；

4.若有帐号密码变动时，需要重新制作数据库文件并重新同步Master/Slave。

Nis Client

1.NIS client 若有登入需求时，会先查询其本机的 /etc/passwd, /etc/shadow 等档案； 

2.若在 NIS Client 本机找不到相关的账号数据，才开始向整个 NIS 网域的主机广播查询； 

3.每部 NIS server (不论 master/slave) 都可以响应，基本上是『先响应者优先』。
```

### NIS环境组件

```shell
1.Nis Master Server：将文件建成数据库，并提供给Slave Server来更新；

2.Nis Slave Server：以Master Server的数据库作为本身的数据库来源；

3.Nis Client：向Master/Slave 请求登陆者的验证数据。
```

### NIS Server信息提供

| server端文件名    | 内容                                               |
| ----------------- | -------------------------------------------------- |
| /etc/passwd       | 提供用户账号、UID、GID、家目录所在、Shell 等等     |
| /etc/group        | 提供群组数据以及 GID 的对应，还有该群组的加入人员  |
| /etc/hosts        | 主机名与 IP 的对应，常用于 private IP 的主机名对应 |
| /etc/services     | 每一种服务 (daemons) 所对应的端口号 (port number)  |
| /etc/protocols    | 基础的 TCP/IP 封包协定，如 TCP, UDP, ICMP 等       |
| /etc/rpc          | 每种 RPC 服务器所对应的程序号码                    |
| /var/yp/ypservers | NIS 服务器所提供的数据库                           |

### NIS环境介绍

| 操作系统  | Centos7.5                                           |
| --------- | --------------------------------------------------- |
| NIS服务端 | 172.26.159.168                                      |
| NIS客户端 | 172.26.159.169                                      |
| 需求      | 服务端与客户端hosts文件均有对方主机名与ip映射记录。 |

### NIS服务端配置

```shell
1,安装NIS Server以及相关软件包
yum -y install ypserv,ypbind,yp-tool

2,设定NIS网域名称
NIS是会分domain name来分辨不同的账号密码工具的，因此必须要在服务器端指定NIS域名称
[root@node5 yp]# cat /etc/sysconfig/network
# Created by anaconda
NISDOMAIN=vbirdnis
YPSERV_ARGS="-p 1011"

3,增加开机自动进入NIS域
echo "/bin/nisdomainname vbird" >> /etc/rc.d/rc.local

4,NIS服务器访问权限设置
配置文件:[root@node5 yp]# cat /etc/ypserv.conf 
# Host                     : Domain  : Map              : Security 
#
# *                        : *       : passwd.byname    : port 
# *                        : *       : passwd.byuid     : port

# Not everybody should see the shadow passwords, not secure, since
# under MSDOG everbody is root and can access ports < 1024 !!!
*                          : *       : shadow.byname    : port
*                          : *       : passwd.adjunct.byname : port

# If you comment out the next rule, ypserv and rpc.ypxfrd will
# look for YP_SECURE and YP_AUTHDES in the maps. This will make
# the security check a little bit slower, but you only have to
# change the keys on the master server, not the configuration files
# on each NIS server.
# If you have maps with YP_SECURE or YP_AUTHDES, you should create
# a rule for them above, that's much faster.
# *                        : *       : *                : none
127.0.0.1/255.255.255.0  : * : * : none
172.26.159.0/255.255.255.0 : * : * : none
*                           : * : * : deny
#冒号为分隔符
Host：指定客户端，可以指定具体IP地址，也可以指定一个网段

Domain：设置NIS域名，这里的NIS域名和DNS中的域名并没有关系，两者是两套不同系统。

Map：设置可用数据库名称，可以用“*”代替所有数据库

Security：安全性设置。主要有none、port和deny三种参数设置。

none：没有任何安全限制，可以连接NIS服务器。

port：只允许小于1024以下的端口连接NIS服务器。

deny：拒绝连接NIS服务器。

另外，ypserv.conf文件是逐行解释执行，所以要注意设置顺序

5,启动和观察相关服务
systemctl start rpcbind
systemctl start ypserv

6,验证是否启动成功
rpcinfo -u localhost ypserv

7,建立数据库
执行命令 /usr/lib64/yp/ypinit -m建立数据库，将nis服务器端的用户信息导入，当用户信息更新时，也需重新执行该命令更新数据库。

[root@node5 yp]# /usr/lib64/yp/ypinit -m
At this point, we have to construct a list of the hosts which will run NIS
servers.  node5.vbird is in the list of NIS server hosts.  Please continue to add
the names for the other hosts, one per line.  When you are done with the
list, type a <control D>.
        next host to add:  node5.vbird
         next host to add: (可以加在NIS域当中的client)可以选择同步的NIS客户端

8,第一次建立完数据库之后需要重启服务器
systemctl start ypserv

9,当在server端建立信账户之后需要进行同步
cd /var/yp/ && make
```

### NIS客户端配置

```shell
1,配置NIS客户端相关软件包
yum -y install ypbind yp-tool

2,在网络加入NIS域
vi /etc/sysconfig/network
添加NISDOMAIN=vbird

3,添加开机自启动
echo "/bin/nisdomainname >> vbird" >> /etc/tc.d/rc.local

4,修改用户密码认证的顺序文件
vi /etc/nsswitch.conf 
passwd:     files nis
shadow:     files nis
group:      files nis

5,修改客户端指向NISserver
vi /etc/yp.conf
domain vbird server 192.168.0.243

6,修改系统认证文件
vi /etc/sysconfig/authconfig
USENIS=yes

vi /etc/pam.d/system-auth(将nis加上)
password sufficient pam_unix.so sha512 shadow nis nullok try_first_pass use_authtok

7,重启服务
systemctl start rpcbind
systemctl start ypbind
systemctl enable rpcbind
systemctl enable ypbind
```