---
title: "Linux Bash"
weight: 1
bookToc: false
---

# Linux Bash

主要介绍常用`Bash shell`命令

## 进程
基于进程名`processname`查找
```
ps aux | grep <processname>
```  
基于进程名`processname`查找，结果中去掉`grep`命令本身
```
ps aux | grep -v grep | grep kubelet
```
某个<PID>进程资源占用
```
htop -p <PID>
```
终止进程
```
kill -9 <PID>
```

{{< expand "singal列表" "..." >}}
```
kill -l
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```
{{< /expand >}}

{{< hint info >}}
[**ps -ef 和 ps aux的区别**](https://askubuntu.com/questions/129962/ps-ef-vs-ps-aux)  
-e和ax为两种syntax风格，单他们的含义是相同的，即展示所有用户的全部进程信息

-e为标准POSIX风格，ax为BSD风格

ps -f输出的列包含，-f = full
```
UID        PID  PPID  C STIME TTY          TIME CMD
```

ps u输出的列包含，u = user-oriented  
```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
```

{{< /hint >}}

## 文件
按文件大小排序显示
```
ls -al
```
按文件更新时间排序显示（上旧下新），去掉参数`r`为上新下旧
```
ls -alrt
```
基于文件/目录名`filena*`查询文件/目录
```
find . -name filena*
```
基于目录名di*查询目录
```
find . -name di* -type d
```
基于文件内容`search term`查询文件
```
grep -ir 'search term' .
```
找到文件`/path/to/file`当前被哪些进程打开
```
lsof /path/to/file
```

## 网络
打开了`5432`端口的进程
```
lsof -i :5432
```
`kubelet`进程打开了哪些端口
```
lsof -i -P -n | grep kubelet
```
关闭防火墙
{{< tabs "close firewall" >}}
{{< tab "CentOS6及以下" >}}
```
sudo service iptables stop  # CentOS 6及以下系统
sudo systemctl stop iptables  # CentOS 6及以下系统
```
{{< /tab >}}
{{< tab "CentOS7及以上" >}}
```
sudo service firewalld stop  # CentOS 7及以上系统
sudo systemctl stop firewalld  # CentOS 7及以上系统
```
{{< /tab >}}
{{< tab "Ubuntu" >}}
```
sudo ufw disable  # Ubuntu系统
```
{{< /tab >}}
{{< /tabs >}}
清空`iptables`
```
sudo iptables -F  # 清除所有防火墙规则
sudo iptables -X  # 删除所有用户自定义链
sudo iptables -Z  # 将所有计数器清零
```
{{< hint info >}}
**iptables计数器是什么**  
iptables计数器是指记录iptables规则操作的数据包或字节数的计数器。当一个数据报匹配了某个规则时，相关的计数器就会被更新，以便后续统计和分析。iptables计数器分为两种，即包计数器和字节计数器。
{{< /hint >}}

iptables黑名单，限制访问来源访问某端口，如禁止IP段192.168.1.0/24访问网络服务的FTP端口21
```
iptables -I INPUT -s 192.168.1.0/24 -p tcp --dport 21 -j DROP 
```
iptables白名单，只允许来自特定来源访问某端口，如只允许IP段192.168.1.0/24访问网络服务的FTP端口21
```
iptables -I INPUT -s 192.168.1.0/24 -p tcp --dport 21 -j ACCEPT
iptables -P INPUT DROP # --policy -P chain target / Change policy on chain to target
```
iptables规则持久化
{{< tabs "iptables persistent" >}}
{{< tab "CentOS6及以下" >}}
```
yum install -y iptables-services
service iptables save # 保存到/etc/sysconfig/iptables
service iptables start
```
{{< /tab >}}
{{< tab "CentOS7及以上" >}}
```
sudo yum install -y iptables-services
sudo iptables -A INPUT -p icmp -j DROP
sudo iptables -L
sudo iptables-save > /etc/sysconfig/iptables # 手动恢复 iptables-restore < /etc/sysconfig/iptables
sudo systemctl start iptables
sudo systemctl enable iptables
```
{{< /tab >}}
{{< tab "Ubuntu" >}}
```
sudo apt-get update
sudo apt-get install iptables-persistent
sudo iptables -A INPUT -p icmp -j DROP
sudo iptables -L
sudo netfilter-persistent save # 保存在/etc/iptables/rules.v4和/etc/iptables/rules.v6
sudo systemctl enable netfilter-persistent
```
{{< /tab >}}
{{< /tabs >}}