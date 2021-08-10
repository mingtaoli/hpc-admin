# 高性能计算集群安装记录

## CentOS 操作系统安装

### 下载CentOS 7.9的ISO文件
到CentOS的[官方站点](https://centos.org/)下载CentOS 7.9-2009的DVD.iso文件，我们也可以到学校的[镜像站点](https://mirrors.xjtu.edu.cn/centos/7.9.2009/isos/x86_64/)下载。下载后通过sha256校验和（windows 10 下直接右键->CRC SHA->SHA-256查看）确保文件下载正确。

### 制作系统启动U盘
Linux启动盘制作可以选择多个软件，rufus是Ubuntu官方推荐的软件之一。到[rufus官方](http://rufus.ie/zh/)下载rufus，并安装。然后用清空的U盘制作系统启动盘，操作过程可以参考[这个链接](https://jingyan.baidu.com/article/0a52e3f48ad2b8bf62ed7236.html)。

### 设置从U盘启动
插入U盘，开机时按Delete键，进入BIOS设置界面。使用方向键，选择Boot菜单栏，在Boot Option #1 处选择从UEFI U盘启动，然后选择Exit菜单栏，选中 Save Changes and Exit。

### 启动CentOS安装程序
从U盘启动后选择 Install CentOS 7 ,回车。等待一段时间后，出现 WELCOME TO CENTOS LINUX 7 界面，语言选择英语，时间与地区选择 Asia Shanghai ，语言、键盘选择English(US)。

### 网络设置
在NETWORK & HOST NAME 设置中，选中外网网卡，手动设置ipv4，设置ip为x.x.x.x， 并按照给定的网络设置要求设置好子网掩码、网关、域名服务器。选中内网网卡，设置ip为192.168.20.1。
hostname 设置为mu01。计算节点内网ip顺序设置为192.168.20.11, 192.168.20.12, ...，设置网关为192.168.20.1。而相应的hostname设置为cu01, cu02, ...。
ip等的设置也可以在安装完成重启之后使用nmtui(Text User Interface for controlling NetworkManager)命令进行。
也可以待全部安装完之后使用hostnamectl设置hostname：
```shell
#hostnamectl set-hostname cu01
```
### 硬盘分区
在INSTALLATION DESTINATION页面，使用manual分区，按照如下的设置分区。
```shell
/boot/efi 1024MiB EFI分区
/boot     2048MiB ext4分区
/         259GiB  ext4分区
swap      16GiB   swap分区
```
接下来设置root密码，添加一个普通用户并设置密码。
等待安装完成重启，CentOS操作系统安装完成。在所有的计算节点重复此过程。如果计算节点特别多，则要考虑无人值守安装了，参考安装后/root目录生成的anaconda-ks.cfg文件。

## 设置yum源
在管理节点修改/etc/yum.repos.d/下的文件，可以参考CentOS镜像站点的说明，比如[这个](https://mirrors.xjtu.edu.cn/help/centos.html)。

然后将管理节点/etc/yum.repos.d/目录中的文件分发至各计算节点的对应位置
```shell
#scp /etc/yum.repos.d/* root@cu01:/etc/yum.repos.d/
```
yum几个常用的命令：
```shell
#yum install packagename #安装软件包packagename
#yum erase packagename #删除软件包packagename
#yum search string #搜索软件描述中含string的软件包
#yum provides *string #搜索哪个软件包提供文件名中含string的文件
#yum list packagename #查看软件包packagename简要信息：
#yum info packagename #查看软件包packagename详细信息
#yum list installed #列出已安装软件包
```
## 设置/etc/hosts
在所有的节点设置/etc/hosts内容如下：
```shell
#
# /etc/hosts file for 49 hpc cluster
# created by Mingtao Li (mingtao@xjtu.edu.cn)
#
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

# head node
192.168.20.1     49mu01 mu01 node00 admin1 io1 logon1

# compute nodes
192.168.20.11    cu01 node01
192.168.20.12    cu02 node02

# InfiniBand ip for head node
10.0.0.1         ib49mu01 ibmu01 ibnode00 ibadmin1 ibio1 iblogon1

# InfiniBand ip for compute nodes
10.0.0.11        ibcu01 ibnode01
10.0.0.12        ibcu02 ibnode02
```
可以在管理节点设置好/etc/hosts，然后使用如下命令同步到各计算节点：
```shell
#scp /etc/hosts cu01:/etc/
```
## 设置root在节点间无密码跳转
使用ssh-keygen生成秘钥公钥对，把公钥复制成authorized_keys，然后把.ssh目录的内容分发到每个节点。
```shell
#ssh-keygen
#cd .ssh
#cp id_rsa.pub authorized_keys
#scp * cu01:/root/.ssh
#scp ...
```
## 防火墙与SELinux设置
在所有的计算节点关闭防火墙：
```shell
#systemctl disable firewalld
#systemctl stop firewalld
```
在所有的计算节点允许用户使用nfs作为home目录：
```shell
#setsebool -P use_nfs_home_dirs 1
```
管理结点我们在网络设置中已经把网卡分别设置为intrenal和external了。在internal上，我们开启我们需要的服务。在external上只开启sshd等有限的服务。

## 安装设置ntp
计算集群的时间设置要统一。如果服务器时间设置不对，yum install等是很可能会失败的，我曾经踩坑无数。所以我们需要使用ntp。在所有的节点上使用如下命令安装并启用ntp
```shell
#yum install ntp
#systemctl enable ntpd
#systemctl start ntpd
```
我们把管理节点的/etc/ntp.conf设置如下：
```shell
#cat /etc/ntp.conf
driftfile /var/lib/ntp/drift
restrict default nomodify notrap nopeer noquery
restrict 127.0.0.1 

restrict 192.168.20.0 mask 255.255.255.0 nomodify notrap

server 0.cn.pool.ntp.org
server 0.centos.pool.ntp.org iburst
server 1.centos.pool.ntp.org iburst
server 2.centos.pool.ntp.org iburst
server 3.centos.pool.ntp.org iburst

server 127.127.1.0
fudge  127.127.1.0 stratum 10

includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
disable monitor
```
而计算节点的/etc/ntp.conf则如下：
```shell
#cat /etc/ntp.conf
driftfile /var/lib/ntp/drift
restrict 127.0.0.1

server 192.168.20.1
server 127.127.1.0

fudge  127.127.1.0 stratum 5

includefile /etc/ntp/crypto/pw
keys /etc/ntp/keys
disable monitor
```
最后在所有的节点通过如下命令重启ntpd并查看时间同步信息：
```shell
#systemctl restart ntpd
#systemctl status ntpd
#timedatectl
```
## 安装设置NFS
NFS就是网络文件系统，通俗的讲，就是IO节点启用“网盘”，给计算节点提供存储服务。在我们的小型集群中，管理节点同时也是IO节点。首先使用如下命令安装nfs-server：
```shell
#yum -y install nfs-utils
```
接下来，边界/etc/exports，设置共享哪些目录出去作为网盘，以及如何共享：
```shell
#
# /etc/exports file for 49 hpc cluster on mu01
# created by Mingtao Li (mingtao@xjtu.edu.cn)
#
/public 192.168.20.0/24(rw,no_root_squash,no_subtree_check,sync) 10.0.0.0/24(rw,no_root_squash,no_subtree_check,sync)
```
然后使用如下命令启动nfs，允许nfs开机自动启动，并查看nfs服务状态：
```shell
#systemctl start nfs
#systemctl enable nfs
#systemctl status nfs
```
可以修改/etc/exports，然后刷新NFS设置使其生效：exportfs -ra
使用以下的命令查看NFS状态：
```shell
#showmount -e
#exportfs -v
```
计算节点是nfs-client，同样通过如下命令安装：
```shell
#yum -y install nfs-utils
```
然后我们可以通过如下命令手动挂载/public目录：
```shell
#mkdir /public
#mount 192.168.20.1:/public /public
```
为了开机自动挂载nfs，我们可以设置/etc/fstab实现，也可以使用/etc/rc.local调用nfs.local实现。为了避免fstab出错系统无法启动的情况出现，我们使用rc.local的方式。
管理节点的内容如下：
```shell
#chmod +x /etc/rc.d/rc.local
#cat /etc/rc.local
touch /var/lock/subsys/local
source /etc/nfs.local
#cat /etc/nfs.local
#echo rdma 20049 > /proc/fs/nfsd/portlist
echo tcp 2049 > /proc/fs/nfsd/portlist
echo udp 2049 > /proc/fs/nfsd/portlist
mount --rbind /public/home /home
```
计算节点的内容如下：
```shell
#chmod +x /etc/rc.d/rc.local
#cat /etc/rc.local
touch /var/lock/subsys/local
source /etc/nfs.local
#cat /etc/nfs.local
mount 10.0.0.1:/public /public/
#mount -o rdma,port=20049 10.0.0.1:/public /public/
mount --rbind /public/home /home
```
然后我们可以使用如下命令查看挂载情况：
```shell
#mount
```
这里说明一下，rdma 20049这些和nfsrdma有关，我们后面装好了InfiniBand网络再说。而mount --rbind /public/home /home是为了把/public/home映射成/home目录，并且保证所有的计算节点的/home来自于网络文件系统。

## 安装Infiniband驱动
我们集群的计算网络是InfiniBand高速网络。以下是InfiniBand卡的信息：
```shell
  Device Type:      ConnectX2
  Part Number:      MHQH19B-XTR_A1-A3
  Description:      ConnectX-2 VPI adapter card; single-port 40Gb/s QSFP; PCIe2.0 x8 5.0GT/s; tall bracket; RoHS R6
  PSID:             MT_0D90110009
  PCI Device Name:  83:00.0
  Port1 GUID:       0002c9030028207f
  Port2 MAC:        0002c928207f
  Versions:         Current        Available
     FW             2.9.1000       2.9.1000
```
InfiniBand卡需要安装驱动。我们到[Mellanox官方站点](https://www.mellanox.com/products/infiniband-drivers/linux/mlnx_ofed)下载驱动。这里需要根据驱动程序版本、操作系统发行版版本及架构选择合适的文件下载，可以是iso文件也可以是tgz文件，前者需要挂载使用，后者需要解压使用。由于我们的InfiniBand卡比较旧，我们选择LTS版本下载。最后我们下载的文件是MLNX_OFED_LINUX-4.9-3.1.5.0-rhel7.9-x86_64.tgz。

我们把文件上传到管理节点mu01的/public/soft目录，解压：
```shell
#tar zvxf MLNX_OFED_LINUX-4.9-3.1.5.0-rhel7.9-x86_64.tgz
```
尝试安装：
```shell
#cd MLNX_OFED_LINUX-4.9-3.1.5.0-rhel7.9-x86_64
#./mlnxofedinstall --help
#./mlnxofedinstall
```
发现缺少perl，而且装完perl之后，还缺少一些依赖，我们先把这些装上：
```shell
#yum install -y perl
#yum install -y pciutils gtk2 atk cairo gcc-gfortran libxml2-python tcsh libnl lsof tcl tk
```
而且，由于我们要启用nfsrdma，所以最后的安装命令如下：
```shell
#./mlnxofedinstall.sh --with-nfsrdma
```
安装完之后启用openibd，并且在管理节点上启用opensmd
```shell
#systemctl start openibd
#systemctl enable opensmd
#systemctl start opensmd
```
使用ibstat查看InfiniBand卡状态。

在所有的计算节点上都这样安装好InfiniBand卡驱动。但是只在管理节点上启用opensmd。由于我们已经设置好了nfs，所以计算节点也只需要进入/public/soft去操作即可。

## 设置Infiniband网络
在所有的节点（包括管理节点和计算节点）把/etc/sysconfig/network-scripts/ifcfg-ib0内容设置如下（注意保留原始的uuid，而ip根据不同的节点设置）：
```shell
#cat /etc/sysconfig/network-scripts/ifcfg-ib0
CONNECTED_MODE=no
TYPE=InfiniBand
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=no
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=ib0
UUID=xxxxyyydfasdfadfafdsfasfadfsffdadaf
DEVICE=ib0
ONBOOT=yes
IPADDR=10.0.0.1
PREFIX=24
DNS=8.8.8.8
ZONE=internal
```
也可以使用nmtui完成ip over IB的设置。
然后使用如下命令使以上设置生效并确认：
```shell
#systemctl restart network
#ip addr show
```
## 修改NFS走rdma

nfsrdma曾经从Mellanox的OFED驱动中去掉，后来又加回来了，但是不是默认安装。我们在安装驱动时使用--with-nfsrdma就已经安装了相应的模块。我们参考[这里](https://community.mellanox.com/s/article/howto-configure-nfs-over-rdma--roce-x)进行nfsrdma的设置。

简要的说，就是nfs服务器需要加载svcrdma内核模块：
```shell
#modprobe svcrdma
```
然后，让nfs监听rdma 20049端口：
```shell
mu01#echo rdma 20049 > /proc/fs/nfsd/portlist
mu01#cat /proc/fs/nfsd/portlist
udp 2049
udp 2049
tcp 2049
tcp 2049
rdma 20049
rdma 20049
```
nfs客户端需要加载xprtrdma内核模块：
```shell
#modprobe xprtrdma
```
可以手动尝试rdma挂载：
```shell
#umount /public
#mount -o rdma,port=20049 mu01:/public /public
```
可以按照上面给出的[链接](https://community.mellanox.com/s/article/howto-configure-nfs-over-rdma--roce-x)检查rdma读写速度。
```shell
# fio --rw=randread --bs=64k --numjobs=4 --iodepth=8 --runtime=30 --time_based --loops=1 --ioengine=libaio --direct=1 --invalidate=1 --fsync_on_close=1 --randrepeat=1 --norandommap --exitall --name task1 --filename=/public/1.txt --size=10000000
```
为了让管理节点和计算节点都开机自动处理nfs的问题，我们需要它们开机自动加载相应的内核模块，并设置nfs问题，我们在管理节点和计算节点都设置了/etc/modules-load.d/rpcrdma.conf：
```shell
#cat /etc/modules-load.d/rpcrdma.conf
rpcrdma
svcrdma
xprtrdma
```
并且把管理节点的/etc/nfs.local设置如下：
```shell
#cat /etc/nfs.local
echo rdma 20049 > /proc/fs/nfsd/portlist
echo tcp 2049 > /proc/fs/nfsd/portlist
echo udp 2049 > /proc/fs/nfsd/portlist
mount --rbind /public/home /home
```
把计算节点的/etc/nfs.local设置如下：
```shell
#cat /etc/nfs.local
mount -o rdma,port=20049 10.0.0.1:/public /public/
mount --rbind /public/home /home
```
我们重启所有的节点检查确保正确。我们需要管理节点启动了再启动计算节点，使用如下的命令：
```shell
#for i in cu01 cu02; do ssh $i "shutdown -r +5"; done
#shutdown -r now
```

## 管理节点安装ldap
现在要进行集群用户管理设置。集群的用户同步可以使用手动、nis或者ldap等不同的方式。

```shell
#yum install openldap openldap-servers openldap-clients
```
以下参考[这个链接](http://blog.chinaunix.net/uid-21926461-id-5676013.html)进行。核心思想就是建立组织数据库，添加用户的时候，把添加的用户的passwd、shadow和group中的记录都更新到ldap的数据库中去，客户端读取数据库获得用户信息。集群管理员可以参考如下的文件信息：
```shell
# cat ldapset.sh
ldapadd -Y EXTERNAL -H ldapi:/// -f 00changepwd.ldif

ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/nis.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/collective.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/corba.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/duaconf.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/dyngroup.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/java.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/misc.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/openldap.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/pmi.ldif
ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/ppolicy.ldif

ldapmodify -Y EXTERNAL -H ldapi:/// -f 01changedomain.ldif

ldapadd -Q -Y EXTERNAL -H ldapi:/// -f 02add-memberof.ldif
ldapmodify -Q -Y EXTERNAL -H ldapi:/// -f 03refint1.ldif
ldapadd -Q -Y EXTERNAL -H ldapi:/// -f 04refint2.ldif

ldapadd -x -D cn=admin,dc=49mu01,dc=org -W -f 05base.ldif
```
添加用户时，首先使用groupadd和useradd添加用户，然后使用passwd设置密码，接下来使用migrationtools（在/usr/share/migrationtools/下有很多命令）生成对应的ldif文件，使用ldapadd更新进ldap数据库。

```shell
#useradd -m -d /home/username -s /bin/bash username
#passwd username
# cat useradd.sh
getent passwd |grep username  > /root/users
getent shadow |grep username  > /root/shadow
getent group |grep username > /root/groups
./migrate_passwd.pl /root/users > /root/users.ldif
./migrate_group.pl /root/groups > /root/groups.ldif
ldapadd -x -W -D "cn=admin,dc=mydomain,dc=com" -f users.ldif
ldapadd -x -W -D "cn=admin,dc=mydomain,dc=com" -f groups.ldif
```
因为我们集群机子不多，手动同步用户稳定可靠，工作量也不算很大，以下内容从略。

## 计算节点使用ldap进行用户授权
计算节点安装nss-pam-ldapd：
```shell
#yum install nss-pam-ldapd
```
使用authconfig-tui进行用户授权设置，使问题简化：
```shell
#authconfig-tui
```
USE LDAP->Use LDAP Authentication->ldap://192.168.20.1/->dc=mydomain,dc=com->ok

使用如下命令检查ldap是否生效：
```shell
#getent passwd|grep myusername
#getent group|grep myusername
#getent shadow|grep myusername
```
## 安装munge
安装munge和slurm网上有很多参考，比如[这个](https://blog.csdn.net/qq_34149581/article/details/101902935)。我们直接从源码安装。

先把依赖openssl装上
```shell
#yum -y install openssl openssl-devel
```
然后到[mung的老巢](https://github.com/dun/munge)去下载最新版的munge源码，解压，编译，安装：
```shell
#tar xJf munge-0.5.14.tar.xz 
#cd munge-0.5.14
#./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var  --with-crypto-lib=openssl --runstatedir=/run --with-systemdunitdir=/usr/lib/systemd/system
#make
#make check
#make install
```

接下来创建munge.key，其内容在32个字节以上，文件权限为400。
```shell
#echo -n "Hello.Kitty.This is your munge key." | sha1sum | cut -d' ' -f1 > munge.key
```
使用如下命令启用munge：
```shell
#cat munge.sh
export MUNGEUSER=500
groupadd -g $MUNGEUSER munge
useradd  -m -c "MUNGE Uid 500 Gid 500" -d /var/lib/munge -u $MUNGEUSER -g munge  -s /sbin/nologin munge

mkdir -p /var/{run,lib,log}/munge
chown -R munge:munge /var/{run,lib,log}/munge
chmod 711 /var/lib/munge
chmod 700 /var/log/munge
chmod 755 /var/run/munge

cp munge.key /etc/munge/

chown -R munge:munge /etc/munge/
chmod 400 /etc/munge/munge.key

systemctl enable munge
yum install readline-devel -y -q #计算节点需要安装缺少的依赖
systemctl start munge
systemctl status munge

munge -n
munge -n | unmunge
```
在计算节点也安装好munge之后使用如下命令测试
```shell
munge -n |ssh cu01 unmunge
```

## 安装slurm

我们使用mariadb作为slurm的数据库：
```shell
#yum install mariadb
```
我们直接从源码安装slurm。到[slurm的老巢](https://slurm.schedmd.com/download.html)去下载最新版的slurm源码，解压，编译，安装：
```shell
#tar xjvf slurm-20.11.8.tar.bz2 
#cd slurm-20.11.8
#./configure --prefix=/public/clustersoft/slurm/slurm20.11.8 --sysconfdir=/public/clustersoft/slurm/slurm20.11.8/etc --localstatedir=/var --enable-pam --enable-memory-leak-debug --enable-salloc-kill-cmd --with-zlib --with-mysql_config=/usr/bin/ --with-munge --with-libcurl
#make
#make install
```

这里管理节点安装的目录是在nfs上的。所以计算节点不需要再安装。但是要进行相应的设置。
首先设置mariadb：
```shell
#cat /etc/my.cnf
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# Settings user and group are ignored when systemd is used.
# If you need to run mysqld under a different user or group,
# customize your systemd unit file for mariadb according to the
# instructions in http://fedoraproject.org/wiki/Systemd

character-set-server=utf8
collation-server=utf8_general_ci
innodb_buffer_pool_size=2G
innodb_log_file_size=2G
innodb_lock_wait_timeout=900

[mysqld_safe]
log-error=/var/log/mariadb/mariadb.log
pid-file=/var/run/mariadb/mariadb.pid


[client]
default-character-set=utf8

[mysql]
default-character-set=utf8

#
# include all files from the config directory
#
!includedir /etc/my.cnf.d
```
创建数据库slurm_acct_db：
```shell
#cat slurm_acct_db.sql
create database slurm_acct_db;
create user 'slurm'@'mu01';
set password for 'slurm'@'mu01' = password('mypasswd');
grant usage on *.* to 'slurm'@'mu01';
grant all privileges on slurm_acct_db.* to 'slurm'@'mu01';
flush privileges;
```
确保mariadb开机就运行正确。
```shell
#systemctl enable mariadb
```

创建用户slurm：
```shell
export SLURMUSER=501
groupadd -g $SLURMUSER slurm
useradd  -m -c "SLURM workload manager Uid 501 Gid 501" -d /var/lib/slurm -u $SLURMUSER -g slurm  -s /bin/bash slurm
```
创建/public/clustersoft/slurm/slurm20.11.8/etc/slurm.conf、cgroup.conf和slurmdbd.conf文件：
```shell
#cat cgoup.conf
CgroupAutomount=yes
CgroupReleaseAgentDir=/tmp/slurm
TaskAffinity=no
ConstrainCores=yes
ConstrainRAMSpace=yes
MaxRAMPercent=98
AllowedRAMSpace=96
#cat slurm.conf
ClusterName=mu01
SlurmctldHost=mu01

SlurmUser=slurm
SlurmdUser=root
SlurmctldPort=6817
SlurmdPort=6818
AuthType=auth/munge

StateSaveLocation=/public/clustersoft/slurm/slurm20.11.8/state
SlurmdSpoolDir=/var/spool/slurm/slurmd

SwitchType=switch/none
TaskPlugin=task/cgroup
MpiDefault=none
SlurmctldPidFile=/var/run/slurm/slurmctld.pid
SlurmdPidFile=/var/run/slurm/slurmd.pid
ProctrackType=proctrack/cgroup

FirstJobId=00001
ReturnToService=2
MaxJobCount=50000

SlurmctldTimeout=300
SlurmdTimeout=300
InactiveLimit=0
MinJobAge=300
KillWait=30
Waittime=0

SchedulerType=sched/backfill
SelectType=select/linear

SlurmctldDebug=info
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdDebug=info
SlurmdLogFile=/var/log/slurm/slurmd.log

JobCompType=jobcomp/none

JobAcctGatherType=jobacct_gather/linux
JobAcctGatherFrequency=30
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=mu01
AccountingStoragePass=/var/run/munge/munge.socket.2
AccountingStorageUser=slurm
AcctGatherNodeFreq=180
AccountingStoreJobComment=YES

NodeName=mu01  NodeAddr=192.168.20.1 CPUs=12 Procs=12 RealMemory=8192 Sockets=2 CoresPerSocket=6 ThreadsPerCore=1 State=DOWN
NodeName=cu01  NodeAddr=192.168.20.11 CPUs=12 Procs=12 RealMemory=12288 Sockets=2 CoresPerSocket=6 ThreadsPerCore=1 State=DOWN
NodeName=cu02  NodeAddr=192.168.20.12 CPUs=12 Procs=12 RealMemory=12288 Sockets=2 CoresPerSocket=6 ThreadsPerCore=1 State=DOWN

PartitionName=control Nodes=mu01 Default=NO MaxTime=INFINITE State=UP
PartitionName=compute Nodes=cu[01-02] Default=YES MaxTime=INFINITE State=UP
#cat slurmdbd.conf
ArchiveJobs=yes
ArchiveDir="/public/clustersoft/slurm/slurm20.11.8/archive/"
ArchiveSteps=yes
AuthType=auth/munge
AuthInfo=/var/run/munge/munge.socket.2
DbdAddr=localhost
DbdHost=localhost
SlurmUser=slurm
MessageTimeout=60
DebugLevel=verbose
DefaultQOS=normal
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/run/slurm/slurmdbd.pid
StorageType=accounting_storage/mysql
StorageHost=localhost
StoragePort=3306
StoragePass=dlgcdxlgjzdsys
StorageUser=slurm
StorageLoc=slurm_acct_db
```
在编译目录slurm-20.11.8中的etc下有slurmdbd.service、slurmctld.servicd和slurmd.service。把它们适当修改，然后拷贝到/usr/lib/systemd/system下。实际上，我们只修改了slurmdbd.service，让它在mariadb之后启动：
```shell
# diff slurmdbd.service /usr/lib/systemd/system
3c3
< After=network.target munge.service
---
> After=network.target munge.service mariadb.service
```

然后建立相应的目录：
```shell
cd /var/lib
mkdir slurm
chown slurm:slurm slurm

cd /var/log
mkdir slurm
chown slurm:slurm slurm

cd /var/run
mkdir slurm
chown slurm:slurm slurm

cd /var/spool
mkdir slurm
chown slurm:slurm slurm
```
在管理节点启动slurmdbd、slurmctld、slurmd：
```shell
#systemctl start slurmdbd
#systemctl status slurmdbd
#systemctl start slurmctld
#systemctl status slurmctld
#systemctl start slurmd
#systemctl status slurmd
```
在计算节点，我们需要启动slurmd。
同样的先创建slurm用户。曾经我们为了省事，munge和slurm用户都从ldap获取，却总是出问题，这是因为systemd启动slurmd的时候，nss-pam-ldapd可能还没有准备好，此时munge和slurm用户还没有，因此出问题。所以我们都改为静态的添加了munge和slurm用户。

另外，slurmd的运行的pid保存在/var/run/slurm/目录下，而这个/var/run是运行时目录，重启就没有/var/run/slurm/了，所以会出错。不管是管理节点还是计算节点都存在这个问题。为此我们在/etc/tmpfiles.d加了一个slurm.conf解决此问题：
```shell
# cat /etc/tmpfiles.d/slurm.conf
d /run/slurm 0750 slurm slurm -
```

还有一个问题，就是计算节点的slurm是在nfs上，需要nfs挂载好了之后启动slurmd。而systemd启动slurmd的时候nfs并不一定就好了。所以我们使用一个checkNFSMount脚本来处理此事：
```shell
# cat /usr/local/bin/checkNFSMount
#!/bin/bash
if [ -d /public/clustersoft ]; then
   exit 0;
else
   sleep 20;
   exit 0;
fi
```
相应的，对slurmd.service也作了修改：
```shell
# cat /usr/lib/systemd/system/slurmd.service
[Unit]
Description=Slurm node daemon
After=munge.service network.target remote-fs.target
#ConditionPathExists=/public/clustersoft/slurm/slurm20.11.8/etc/slurm.conf

[Service]
Type=simple
EnvironmentFile=-/etc/sysconfig/slurmd
ExecStartPre=/usr/local/bin/checkNFSMount #就是此处，检测NFS是否ok
ExecStart=/public/clustersoft/slurm/slurm20.11.8/sbin/slurmd -D $SLURMD_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
LimitNOFILE=131072
LimitMEMLOCK=infinity
LimitSTACK=infinity
Delegate=yes


[Install]
WantedBy=multi-user.target
```
当然呢，计算节点如果是缺少相应的依赖的话，要都给它装上：
```shell
yum install -y -q gcc openssl openssl-devel pam-devel numactl numactl-devel hwloc hwloc-devel lua lua-devel readline-devel rrdtool-devel ncurses-devel gtk2-devel libibmad libibumad perl-Switch perl-ExtUtils-MakeMaker xorg-x11-xauth
```
计算节点启动相应的服务：
```shell
#systemctl start slurmd
#systemctl status slurmd
#cat /var/run/slurm/slurmd.pid
```
我们在所有的节点把slurm的bin目录加到PATH中去：
```shell
# cat /etc/profile.d/slurm.sh
PATH=/public/clustersoft/slurm/slurm20.11.8/bin/:${PATH}
# cat /etc/profile.d/slurm.csh
set path = ( /public/clustersoft/slurm/slurm20.11.8/bin $path )
```

这样，我们就可以使用sinfo、srun、scontrol等命令了。

## 安装intel parallel studio xe

目前intel parallel studio xe已经更换为Intel oneAPI Toolkits，根据操作系统版本和硬件信息在官网查找相应的版本下载。学生可以申请相应的License。

[这里有个脚本](https://github.com/jeffhammond/HPCInfo/blob/master/buildscripts/icc-release.sh)提供了不同的版本的下载。

我们使用的是parallel_studio_xe_2020_update4_cluster_edition.tgz。我们解压之后，安装它：
```shell
tar zvxf parallel_studio_xe_2020_update4_cluster_edition.tgz
cd parallel_studio_xe_2020_update4_cluster_edition
./install.sh
```
缺少依赖的话，先给它装上：
```shell
yum install gtk3 libXScrnSaver kernel-devel-3.10.0-1160.36.2.el7.x86_64 xorg-x11-server-Xorg gcc-c++
```

装好了之后通过以下命令加载：
```shell
source /opt/intel/parallel_studio_xe_2020/psxevars.sh
```
也可以通过以下命令加入到.bashrc：
```shell
echo "source /opt/intel/parallel_studio_xe_2020/psxevars.sh" >> ~/.bashrc
```
## 安装openmpi

OpenMPI是mpi的一个性能优越的实现。我们使用最新的稳定版4.1.1。安装包可以从[openmpi官网](https://www.open-mpi.org/software/ompi/v4.1/)获取。解压之后进行安装：
```shell
tar -zxvf openmpi-4.1.1.tar.gz
cd openmpi-4.1.1/
./configure --prefix=/public/soft/openmpi/4.1.1 --with-knem=/opt/knem-1.1.4.90mlnx1/ --with-mxm=/opt/mellanox/mxm/ --with-verbs --with-slurm CC=icc CXX=icpc FC=ifort
make
make install
```
然后把编译目录的config.log拷贝到安装目录/public/soft/openmpi/4.1.1中，以便大家了解软件安装的选项。

接下来，编辑~/.bashrc加入如下代码：
```shell
MPI_HOME=/public/soft/openmpi/4.1.1
export  PATH=${MPI_HOME}/bin:$PATH
export  LD_LIBRARY_PATH=${MPI_HOME}/lib:$LD_LIBRARY_PATH
export  MANPATH=${MPI_HOME}/share/man:$MANPATH
```

