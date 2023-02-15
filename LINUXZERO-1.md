# Linux中病毒后的排查

日常备份及恢复快照是最简单省心的操作。没有有这方面措施的话，只能靠着经验来一步步梳理与排查了。其他信息参考：

* [为什么 Linux 内核不适合国家防御](https://blog.yurunsoft.com/a/68.html)
* [Linux系统安全隐患及加强安全管理的方法](https://www.cnblogs.com/myphoebe/archive/2011/08/09/2131982.html)
* [bilibili专栏-应急响应专题（Linux应急响应）](https://www.bilibili.com/read/cv17867865/)

推荐：

* Linux安全隐患排查脚本：[al0ne/LinuxCheck](https://github.com/al0ne/LinuxCheck)
* 了解端口号知识：[csdn-计算机常用端口号大全](https://blog.csdn.net/weixin_42828010/article/details/127500199)。


## 排查异常任务及程序

### 查看命令记录与操作日志记录

查看 `cat /root/.bash_history` ，一般情况黑客都会进行`echo > .bash_history`...基本上也看不到什么记录。不过使用"journal"是可以的，个人推测日志是系统守护进程，而且自己试验过，即使使用日志完全清除命令 `rm /var/log/journal/* -rf;systemctl restart systemd-journald`，也依然会在系统留有一些相关的命令记录痕迹。

`journalctl -u sshd.service` 查看ssh服务日志，看下日常ssh被尝试登录的记录。 然后`id username`看下用户uid，`journalctl _UID=0 -n 50`查看50行root用户行为操作记录。目前的当务之急是处理掉异常程序和账户密钥后门，之后的什么的慢慢排查，如果是[维护运营团队组织的分析，处理事件的优先级那另说](https://www.bilibili.com/read/cv17795783)。

参考：[csdn-journalctl -xe命令(系统日志查询)的使用](https://blog.csdn.net/enthan809882/article/details/104551777/)、[cnblogs-linux下的系统服务管理及日志管理](https://www.cnblogs.com/yuzhaokai0523/p/4453094.html)

### 排查当前存在异常进程

**使用`top`打开任务管理器，`kill -9 进程名`。** 参考：[pomit-Linux中的kill与kill -9](http://www.pomit.cn/tr/5063499771865601)。简单说，"-9"这参数就是不给程序收尾的时间，立马强行中止；这样的话，程序无法完成其下一步将要进行的计划。也可以`man`加单个指令，查看使用详情。

**使用`crontab -l`查看所有的定时任务。** 参考：[csdn-阿里云ECS遭挖矿程序攻击解决方法（彻底清除挖矿程序，顺便下载了挖矿程序的脚本）](https://blog.csdn.net/NicolasLearner/article/details/119006769)、[csdn-crontab -r删除后恢复](https://blog.csdn.net/only_cyk/article/details/123550872)。

**将`/var/spool/cron/用户名文件`的备份，覆盖掉感染病毒的主机定时任务文件。** 没有备份文件的话，就只能`cat /var/spool/cron/用户名文件`逐个通过`crontab -e`编辑去删除可疑任务了。

### 异常流量程序及传输端口

`netstat -u -nat`再看看端口网络协议连接状态，结合`lsof -i 4` 对ipv4连接情况进行分析 ，再使用`kill -9`杀掉相关进程。

ps：也可安装网卡流量监测程序并启动，查看异常的传输流量。`yum install -y iftop && iftop -i eth0 -nNP`，为防止蠕虫病毒通过局域网内部传输的可能性，先临时关闭FTP（21）、SMB（139、445）。

参考：[csdn-Linux定位流量异常指南](https://blog.csdn.net/q2365921/article/details/125006136)、[csdn-Linux 命令 | 常用命令 lsof 详解 + 实例](https://blog.csdn.net/nyist_zxp/article/details/115340302)

## 排查可疑文件、服务模块、自启程序

**48小时内被修改的文件 `find ./ -ctime -2`，并找出相关记录中的可疑文件。** 

**`ll /etc/systemd/system/` 查看可疑最近新增的服务模块；`chattr -i`+文件名，使得文件可以rm掉。**

**`cat /etc/rc.local`，查看可疑的自启程序、服务、脚本等**

参考：

* [csdn-linux中查看新增的文件](https://blog.csdn.net/qq_17576885/article/details/121995103)
* [Linux *.service文件详解](https://blog.csdn.net/weixin_44352521/article/details/126679172)
* [腾讯云-Linux之init.d、rc.d文件夹说明](https://cloud.tencent.com/developer/article/1533529)

以及挖矿病毒源码 `sudo mv /tmp/c3pool_miner.service /etc/systemd/system/c3pool_miner.service` 启发。


## 检查账户相关的后门

若是黑客使用了 `echo > /var/log/wtmp &&  echo > /var/log/btmp` ，那么想用`lastlog | head -n 15` 看可疑用户登录记录就没辙了。

### 检查可疑账户

`getent passwd {1000..60000}`，查看除root外的所有用户。`id`+“用户名” 可查看用户所属组，并查看存在特权组的可疑用户 `cat /etc/group|grep wheel`。

删除用户`userdel -r 用户名`；将用户加入到组 `usermod -G wheel user01`。

`echo "SU_WHEEL_ONLY yes" >> /etc/login.defs` 仅限wheel组用户可以sudo特权提升。`vi /etc/pam.d/su` 并检查sudo权限的配置文件，去掉如下指令注释，防止其他用户提权。

```
# auth required pam_wheel.so use_uid
```

### ssh密钥后门

从根目录搜索".ssh"文件夹，看看是否存在可疑的authorized_keys。

可疑点：

* `diff`比对备份的authorized_keys，发现有差异。
* 可疑家目录的用户及其他文件夹，有多余的authorized_key。

```
cd / && find -name .ssh
```

### 密码策略后门

既然已经被入侵，现在应该注意改密码了，以及查看密码策略`vi /etc/pam.d/system-auth`

```
# 失败3次封锁300秒
auth required  pam_faillock.so preauth silent audit deny=3  unlock_time=300 even_deny_root
# 密码验证三次，忽略大小写特殊字符，最小长度1位密码
password requisite pam_pwquality.so try_first_pass local_users_only retry=3
password requisite pam_pwquality.so authtok_type= lcredit=0 ucredit=0 dcredit=0 ocredit=0  minlen=1
```

`chage -l root`检查账户是否存在变更与过期，以及`vi /etc/login.defs`

```
PASS_MIN_DAYS 0 # 两次修改密码的最小间隔时间，0表示可以随时修改账号密码
PASS_MIN_LEN 8 # 密码最小长度，对于root无效。
PASS_WARN_AGE 7 # 密码过期前多少天开始提示。
PASS_MAX_DAYS 99999 # 99999表示永不过期。
```


* [潇湘隐者-Linux账户密码过期安全策略设置](https://www.cnblogs.com/kerrycode/p/5600525.html)
* [myfreax-Linux getent 命令列出所有用户](https://www.myfreax.com/linux-getent-command-to-list-all-users/amp/)
* [csdn-Linux学习笔记之CentOS7的 wheel组](https://blog.csdn.net/kfepiza/article/details/124701762)
* [csdn-wheel用户组 普用户禁止su 到root 用户设置 Linux](https://blog.csdn.net/MrFDd/article/details/118492246)
* [cnblogs-linux中添加一个用户到指定用户组的两种方式，修改一个用户到指定用户组的一种方式](https://www.cnblogs.com/alonely/p/9425327.html)
* [51cto-kdevtmpfsi挖矿病毒清除](https://blog.51cto.com/liuyj/5205391)
* [【实用】防暴力破解服务器ssh登入次数](https://cloud.tencent.com/developer/article/2142596)


## 回顾与学习Linux启动流程

图转自：[Linux启动管理 - /etc/rc.d/rc.local配置文件用法](http://c.biancheng.net/view/1023.html)



参考：

* [cnblogs-Linux启动过程详解](https://www.cnblogs.com/notepi/archive/2013/06/15/3137093.html)
* [csdn-Linux系统启动流程（超详细）](https://blog.csdn.net/shuju1_/article/details/126201364)
* [Linux启动管理 - /etc/rc.d/rc.local配置文件用法](http://c.biancheng.net/view/1023.html)



## 使用排查安全隐患脚本


### 使用clamav杀毒




