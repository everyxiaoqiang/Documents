#Cobbler自动化部署实战
> Version 1.0 2016-06-03
## Cobbler介绍
>Cobbler是一个Linux服务器安装的服务，可以通过网络启动(PXE)的方式来快速安装、重装物理服务器和虚拟机，同时还可以管理DHCP，DNS等。

>Cobbler可以使用命令行方式管理，也提供了基于Web的界面管理工具(cobbler-web)，还提供了API接口，可以方便二次开发使用。

>Cobbler是较早前的kickstart的升级版，优点是比较容易配置，还自带web界面比较易于管理。

>Cobbler内置了一个轻量级配置管理系统，但它也支持和其它配置管理系统集成，如Puppet，暂时不支持SaltStack。

###Cobbler集成的服务

* PXE服务支持
* DHCP服务管理
* DNS服务管理(可选bind,dnsmasq)
* 电源管理
* Kickstart服务支持
* YUM仓库管理
* TFTP(PXE启动时需要)
* Apache(提供kickstart的安装源，并提供定制化的kickstart配置)
##系统环境准备
###环境检查
<pre>
[root@linux-node1 ~]# cat /etc/redhat-release 
CentOS Linux release 7.2.1511 (Core) 
[root@linux-node1 ~]# uname -r
3.10.0-327.18.2.el7.x86_64
[root@linux-node1 ~]# getenforce 
Disabled
[root@linux-node1 ~]# ifconfig eth0|grep -oP "(?<=inet )\S+"
192.168.56.11
[root@linux-node1 ~]# systemctl stop firewalld
[root@linux-node1 ~]# hostname
linux-node1.example.com
</pre>

####注意：/etc/redhat-release 文件一定不要清空！！
> 如果清空，手动添加！
> 
> cat /etc/redhat-release 
> 
> CentOS Linux release 7.2.1511 (Core) 
###配置yum源
<pre>
rpm -ivh  http://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm
</pre>
###时间同步
<pre>
yum -y install ntpdate
ntpdate ntp1.aliyun.com
date
</pre>

##Cobbler安装配置
###yum安装Cobbler软件包(此次有35个包)
<pre>
yum install cobbler cobbler-web pykickstart httpd tftp dhcp xinetd -y
</pre>
###配置tftp
>[root@linux-node1 ~]# vim /etc/xinetd.d/tftp

>修改14行为      disable          = no
###配置http
>[root@linux-node1 ~]# vim /etc/httpd/conf/httpd.conf 
>
> 第96行插入： ServerName 127.0.0.1:80
###启动软件
<pre>
[root@linux-node1 ~]# systemctl start httpd
[root@linux-node1 ~]# systemctl start rsyncd
[root@linux-node1 ~]# systemctl start cobblerd
</pre>
###检查cobbler配置文件
<pre>
[root@linux-node1 ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : The 'server' field in /etc/cobbler/settings must be set to something other than localhost, or kickstarting features will not work.  This should be a resolvable hostname or IP for the boot server as reachable by all machines that will use it.
2 : For PXE to be functional, the 'next_server' field in /etc/cobbler/settings must be set to something other than 127.0.0.1, and should match the IP of the boot server on the PXE network.
3 : some network boot-loaders are missing from /var/lib/cobbler/loaders, you may run 'cobbler get-loaders' to download them, or, if you only want to handle x86/x86_64 netbooting, you may ensure that you have installed a *recent* version of the syslinux package installed and can ignore this message entirely.  Files in this directory, should you want to support all architectures, should include pxelinux.0, menu.c32, elilo.efi, and yaboot. The 'cobbler get-loaders' command is the easiest way to resolve these requirements.
4 : enable and start rsyncd.service with systemctl
5 : debmirror package is not installed, it will be required to manage debian deployments and repositories
6 : The default password used by the sample templates for newly installed machines (default_password_crypted in /etc/cobbler/settings) is still set to 'cobbler' and should be changed, try: "openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'" to generate new one
7 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
[root@linux-node1 ~]# 
</pre>
####解决第一条
<pre>
[root@linux-node1 ~]# sed -i 's/server: 127.0.0.1/server: 192.168.56.11/' /etc/cobbler/settings
</pre>
####解决第二条
<pre>
[root@linux-node1 ~]# sed -i 's/next_server: 127.0.0.1/next_server: 192.168.56.11/' /etc/cobbler/settings
</pre>
####解决第三条
<pre>
[root@linux-node1 ~]# cobbler get-loaders
</pre>
####解决第四条(*没效果，但不影响后续操作)
<pre>
systemctl start rsyncd
</pre>
####第五条，针对debian,与本文不符
####第六条 初始密码太简单
> openssl passwd -1 -salt 'random-phrase-here' 'your-password-here'
<pre>
[root@linux-node1 ~]# openssl passwd -1 -salt 'cobler' 'cobler'
$1$cobler$XJnisBweZJlhL651HxAM00
</pre>
#####将刚才openssl生成的密码，复制到第101行的引号内
* default_password_crypted: "$1$cobler$XJnisBweZJlhL651HxAM00"
####第七条安装cman工具用于电源管理
* yum  -y install cman

###编辑cobbler的dhcp配置文件,指定可分配ip地址范围

* vim /etc/cobbler/dhcp.template
<pre>
 subnet 192.168.56.0 netmask 255.255.255.0 {
 option routers             192.168.56.2;
 option domain-name-servers 192.168.56.2;
 option subnet-mask         255.255.255.0;
 range dynamic-bootp        192.168.56.100 192.168.56.254;
</pre>


<pre>
sed -i 's#manage_dhcp: 0#manage_dhcp: 1#g' /etc/cobbler/settings
</pre>

###重新启动和检查cobbler
<pre>
[root@linux-node1 ~]# systemctl restart cobblerd
[root@linux-node1 ~]# cobbler check
The following are potential configuration items that you may want to fix:

1 : enable and start rsyncd.service with systemctl
2 : debmirror package is not installed, it will be required to manage debian deployments and repositories
3 : fencing tools were not found, and are required to use the (optional) power management features. install cman or fence-agents to use them

Restart cobblerd and then run 'cobbler sync' to apply changes.
[root@linux-node1 ~]# cobbler sync
task started: 2016-06-04_000529_sync
task started (id=Sync, time=Sat Jun  4 00:05:29 2016)
running pre-sync triggers
cleaning trees
removing: /var/lib/tftpboot/grub/images
copying bootloaders
trying hardlink /var/lib/cobbler/loaders/pxelinux.0 -> /var/lib/tftpboot/pxelinux.0
trying hardlink /var/lib/cobbler/loaders/menu.c32 -> /var/lib/tftpboot/menu.c32
trying hardlink /var/lib/cobbler/loaders/yaboot -> /var/lib/tftpboot/yaboot
trying hardlink /usr/share/syslinux/memdisk -> /var/lib/tftpboot/memdisk
trying hardlink /var/lib/cobbler/loaders/grub-x86.efi -> /var/lib/tftpboot/grub/grub-x86.efi
trying hardlink /var/lib/cobbler/loaders/grub-x86_64.efi -> /var/lib/tftpboot/grub/grub-x86_64.efi
copying distros to tftpboot
copying images
generating PXE configuration files
generating PXE menu structure
rendering TFTPD files
generating /etc/xinetd.d/tftp
cleaning link caches
running post-sync triggers
running python triggers from /var/lib/cobbler/triggers/sync/post/*
running python trigger cobbler.modules.sync_post_restart_services
running shell triggers from /var/lib/cobbler/triggers/sync/post/*
running python triggers from /var/lib/cobbler/triggers/change/*
running python trigger cobbler.modules.scm_track
running shell triggers from /var/lib/cobbler/triggers/change/*
*** TASK COMPLETE ***
[root@linux-node1 ~]# 
</pre>

###挂载光驱镜像
<pre>
[root@linux-node1 ~]# mount /dev/cdrom /mnt
mount: /dev/sr0 is write-protected, mounting read-only
</pre>

##Cobbler的命令行管理
###查看命令帮助
<pre>
    [root@linux-node1 ~]# cobbler
    usage
    =====
    cobbler <distro|profile|system|repo|image|mgmtclass|package|file> ... 
            [add|edit|copy|getks*|list|remove|rename|report] [options|--help]
    cobbler <aclsetup|buildiso|import|list|replicate|report|reposync|sync|validateks|version|signature|get-loaders|hardlink> [options|--help]
    [root@linux-node1 ~]# cobbler import --help  # 导入镜像
    Usage: cobbler [options]
    Options:
      -h, --help            show this help message and exit
      --arch=ARCH           OS architecture being imported
      --breed=BREED         the breed being imported
      --os-version=OS_VERSION
                            the version being imported
      --path=PATH           local path or rsync location
      --name=NAME           name, ex 'RHEL-5'
      --available-as=AVAILABLE_AS
                            tree is here, don't mirror
      --kickstart=KICKSTART_FILE
                            assign this kickstart file
      --rsync-flags=RSYNC_FLAGS
                            pass additional flags to rsync
    cobbler check    核对当前设置是否有问题
    cobbler list     列出所有的cobbler元素
    cobbler report   列出元素的详细信息
    cobbler sync     同步配置到数据目录,更改配置最好都要执行下
    cobbler reposync 同步yum仓库
    cobbler distro   查看导入的发行版系统信息
    cobbler system   查看添加的系统信息
    cobbler profile  查看配置信息
</pre>
###导入镜像
<pre>
[root@linux-node1 ~]# cobbler import --path=/mnt/ --name=CentOS-7-x86_64 --arch=x86_64
task started: 2016-06-04_001936_import
task started (id=Media import, time=Sat Jun  4 00:19:36 2016)
Found a candidate signature: breed=redhat, version=rhel6
Found a candidate signature: breed=redhat, version=rhel7
Found a matching signature: breed=redhat, version=rhel7
Adding distros from path /var/www/cobbler/ks_mirror/CentOS-7-x86_64:
creating new distro: CentOS-7-x86_64
trying symlink: /var/www/cobbler/ks_mirror/CentOS-7-x86_64 -> /var/www/cobbler/links/CentOS-7-x86_64
creating new profile: CentOS-7-x86_64
associating repos
checking for rsync repo(s)
checking for rhn repo(s)
checking for yum repo(s)
starting descent into /var/www/cobbler/ks_mirror/CentOS-7-x86_64 for CentOS-7-x86_64
processing repo at : /var/www/cobbler/ks_mirror/CentOS-7-x86_64
need to process repo/comps: /var/www/cobbler/ks_mirror/CentOS-7-x86_64
looking for /var/www/cobbler/ks_mirror/CentOS-7-x86_64/repodata/*comps*.xml
Keeping repodata as-is :/var/www/cobbler/ks_mirror/CentOS-7-x86_64/repodata
*** TASK COMPLETE ***
[root@linux-node1 ~]# 
</pre>


> --path 镜像路径
> 
> --name 为安装源定义一个名字
> 
> --arch 指定安装源是32位、64位、ia64, 目前支持的选项有: x86│x86_64│ia64

* 查看镜像列表
<pre>
[root@linux-node1 ~]# cobbler distro list  
CentOS-7-x86_64
</pre>
镜像存放目录，cobbler会将镜像中的所有安装文件拷贝到本地一份，放在/var/www/cobbler/ks_mirror下的CentOS-7-x86_64目录下。因此/var/www/cobbler目录必须具有足够容纳安装文件的空间。
<pre>
[root@linux-node1 ~]# cd /var/www/cobbler/ks_mirror/
[root@linux-node1 ks_mirror]# ls
CentOS-7-x86_64  config
[root@linux-node1 ks_mirror]# ls CentOS-7-x86_64/
CentOS_BuildTag  GPL       LiveOS    RPM-GPG-KEY-CentOS-7
EFI              images    Packages  RPM-GPG-KEY-CentOS-Testing-7
EULA             isolinux  repodata  TRANS.TBL
</pre>

###指定ks.cfg文件及调整内核参数
* Cobbler的ks.cfg文件存放位置
<pre>
[root@linux-node1 ~]# cd /var/lib/cobbler/kickstarts/
[root@linux-node1 kickstarts]# ls
default.ks        legacy.ks            sample_esx4.ks   sample_old.seed
esxi4-ks.cfg      pxerescue.ks         sample_esxi4.ks  sample.seed
esxi5-ks.cfg      sample_autoyast.xml  sample_esxi5.ks
install_profiles  sample_end.ks        sample.ks
</pre>

####为镜像指定kc.cfg
* [root@linux-node1 kickstarts]# cobbler profile edit --name=CentOS-7-x86_64 --kickstart=/var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg

####内核配置
* cobbler profile edit --name=CentOS-7-x86_64 --kopts='net.ifnames=0 biosdevname=0'

####查看配置
<pre>
[root@linux-node1 kickstarts]# cobbler profile report
Name                           : CentOS-7-x86_64
TFTP Boot Files                : {}
Comment                        : 
DHCP Tag                       : default
Distribution                   : CentOS-7-x86_64
Enable gPXE?                   : 0
Enable PXE Menu?               : 1
Fetchable Files                : {}
Kernel Options                 : {'biosdevname': '0', 'net.ifnames': '0'}
Kernel Options (Post Install)  : {}
Kickstart                      : /var/lib/cobbler/kickstarts/CentOS-7-x86_64.cfg
Kickstart Metadata             : {}
Management Classes             : []
Management Parameters          : <<inherit>>
Name Servers                   : []
Name Servers Search Path       : []
Owners                         : ['admin']
Parent Profile                 : 
Internal proxy                 : 
Red Hat Management Key         : <<inherit>>
Red Hat Management Server      : <<inherit>>
Repos                          : []
Server Override                : <<inherit>>
Template Files                 : {}
Virt Auto Boot                 : 1
Virt Bridge                    : xenbr0
Virt CPUs                      : 1
Virt Disk Driver Type          : raw
Virt File Size(GB)             : 5
Virt Path                      : 
Virt RAM (MB)                  : 512
Virt Type                      : kvm

[root@linux-node1 kickstarts]# 
</pre>

## 注意：每次修改完都要同步一次
[root@linux-node1 ~]# cobbler sync
</pre>
[root@linux-node1 kickstarts]# cobbler sync
task started: 2016-06-04_004840_sync
task started (id=Sync, time=Sat Jun  4 00:48:40 2016)
running pre-sync triggers
cleaning trees
removing: /var/www/cobbler/images/CentOS-7.1-x86_64
removing: /var/www/cobbler/images/CentOS-7-x86_64
removing: /var/lib/tftpboot/pxelinux.cfg/default
removing: /var/lib/tftpboot/grub/images
removing: /var/lib/tftpboot/grub/grub-x86.efi
removing: /var/lib/tftpboot/grub/grub-x86_64.efi
removing: /var/lib/tftpboot/grub/efidefault
removing: /var/lib/tftpboot/images/CentOS-7.1-x86_64
removing: /var/lib/tftpboot/images/CentOS-7-x86_64
removing: /var/lib/tftpboot/s390x/profile_list
copying bootloaders
trying hardlink /var/lib/cobbler/loaders/grub-x86.efi -> /var/lib/tftpboot/grub/grub-x86.efi
trying hardlink /var/lib/cobbler/loaders/grub-x86_64.efi -> /var/lib/tftpboot/grub/grub-x86_64.efi
copying distros to tftpboot
copying files for distro: CentOS-7-x86_64
trying hardlink /var/www/cobbler/ks_mirror/CentOS-7-x86_64/images/pxeboot/vmlinuz -> /var/lib/tftpboot/images/CentOS-7-x86_64/vmlinuz
trying hardlink /var/www/cobbler/ks_mirror/CentOS-7-x86_64/images/pxeboot/initrd.img -> /var/lib/tftpboot/images/CentOS-7-x86_64/initrd.img
copying images
generating PXE configuration files
generating PXE menu structure
copying files for distro: CentOS-7-x86_64
trying hardlink /var/www/cobbler/ks_mirror/CentOS-7-x86_64/images/pxeboot/vmlinuz -> /var/www/cobbler/images/CentOS-7-x86_64/vmlinuz
trying hardlink /var/www/cobbler/ks_mirror/CentOS-7-x86_64/images/pxeboot/initrd.img -> /var/www/cobbler/images/CentOS-7-x86_64/initrd.img
Writing template files for CentOS-7-x86_64
rendering TFTPD files
generating /etc/xinetd.d/tftp
processing boot_files for distro: CentOS-7-x86_64
cleaning link caches
running post-sync triggers
running python triggers from /var/lib/cobbler/triggers/sync/post/*
running python trigger cobbler.modules.sync_post_restart_services
running shell triggers from /var/lib/cobbler/triggers/sync/post/*
running python triggers from /var/lib/cobbler/triggers/change/*
running python trigger cobbler.modules.scm_track
running shell triggers from /var/lib/cobbler/triggers/change/*
*** TASK COMPLETE ***
[root@linux-node1 kickstarts]# 
</pre>

##检验结果
###创建一台新的虚拟机，选择与cobbler服务器同一网段，出现如下界面，即成功
![图2](https://github.com/everyxiaoqiang/Documents/raw/master/images/QQ%E5%9B%BE%E7%89%8720160604010026.png)

<pre>
tail /var/log/messages 
Jun  4 00:56:48 linux-node1 dhcpd: DHCPDISCOVER from 00:0c:29:64:10:48 via eth0
Jun  4 00:56:49 linux-node1 dhcpd: DHCPOFFER on 192.168.56.100 to 00:0c:29:64:10:48 via eth0
Jun  4 00:56:50 linux-node1 dhcpd: DHCPREQUEST for 192.168.56.100 (192.168.56.11) from 00:0c:29:64:10:48 via eth0
Jun  4 00:56:50 linux-node1 dhcpd: DHCPACK on 192.168.56.100 to 00:0c:29:64:10:48 via eth0
Jun  4 00:56:50 linux-node1 xinetd[20917]: START: tftp pid=21069 from=192.168.56.100
Jun  4 00:56:50 linux-node1 in.tftpd[21070]: RRQ from 192.168.56.100 filename pxelinux.0
</pre>

##WEB管理
* 输入https://192.168.56.11/cobbler_web，打开管理界面
![图3](https://github.com/everyxiaoqiang/Documents/raw/master/images/QQ%E6%88%AA%E5%9B%BE20160604011934.jpg)
> Username:cobbler
>
> Password:cobbler
###更改web管理密码
<pre>
[root@linux-node1 kickstarts]# htdigest /etc/cobbler/users.digest "Cobbler" cobbler
Changing password for user cobbler in realm Cobbler
New password: 
Re-type new password: 
[root@linux-node1 kickstarts]# 
</pre>
输入两遍123456之后，以后的密码就是123456了