---
title: "Windows CMD"
weight: 2
bookToc: false
---

# Windows CMD

主要介绍`Windows CMD`常用命令

## 文件

切换目录
```
cd C:\Users\zqq\Desktop
```
当前目录
```
cd
```
当前目录中的文件
```
dir /?
dir /o:-s # 从大到小排序
```
查看`1.txt`文件内容
```
type 1.txt # 如果文件太长，可以 more 1.txt
```
删除`1.txt`文件
```
del 1.txt
```
基于文件名`1.txt`查找文件
```
dir /s 1.txt # 包含子目录
```
基于文件内容查找文件
```
findstr /s /i /m /p "xy9527" *
```
{{< hint info >}}
**参数含义**

/s  在当前目录和所有子目录中搜索匹配文件  
/i  指定搜索不分大小写  
/m  如果文件含有匹配项，只打印其文件名  
/p  忽略有不可打印字符的文件
{{< /hint >}}
## 进程

全部进程
```
tasklist /v
```
{{< hint info >}}
**windows下的服务和普通进程有什么区别？**

普通进程和服务处于不同的`Session`。  
服务位于`Session 0`，跟登录的用户无关，可以在`winlogon desktop`级别（用户登录前）启动；  
普通进程处于登录用户的`Session`，只能在`default desktop`级别（用户登录后）启动。
{{< /hint >}}
基于进程名查找进程
```
tasklist | findstr /i "cmd.exe"
```
基于`pid`关闭进程
```
taskkill /pid 1588
```
运行执行文件
```
C:\Windows\System32\ping.exe www.google.com -n 4 # 或者cd C:\Windows\System32 再 ping.exe
```
判断telnet client是否启用
```
telnet
```
启用telnet client
```
dism /online /Enable-Feature /FeatureName:TelnetClient
```
判断rdp是否启用
```
netstat -an | findstr 3389
```
启用rdp（`fDenyTSConnections`默认为1拒绝改为0，修改不用重启）
```
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 0 /f
```
禁用rdp
```
reg add "HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Terminal Server" /v fDenyTSConnections /t REG_DWORD /d 1 /f
```
CMD访问PowerShell
```
powershell
```

## 网络

访问`百度`网页
```
curl www.baidu.com
```
下载文件
```
curl www.baidu.com -o 1.html
```
检测哪个进程打开了`3389`端口
```
netstat -ano | findstr 3389
```
查询防火墙状态
```
netsh advfirewall show allprofiles state
```
关闭防火墙
```
netsh advfirewall set allprofiles state off
```
打开防火墙
```
netsh advfirewall set allprofiles state on
```
{{< hint info >}}
profiles分为3类配置：  
- `publicprofile`：应用于公用网络的配置
- `privateprofile`：应用于专用网络的配置
- `domainprofile`：应用于域网络的配置

在Windows网络中，专用、公用、域三个网络类型是根据网络的安全性和隐私性进行分类的：
- 专用网络：专用网络是指用户在家庭、办公室或内部网络中使用的网络类型
- 公用网络：公用网络是指用户在公共场所或酒店等公共场所中使用的网络类型
- 域网络：域网络是指属于同一组织或公司的多个计算机连接在一起，并受到域管理员的管理和控制

接入一个新的网络时，通常会自动检测网络类型，并将其自动配置为相应的网络类型（专用、公用或域）。  
然而，有时候Windows检测网络类型可能会出现错误，这时需要手动指定网络类型。
{{< /hint >}}
拒绝`10.10.10.10`来源的请求
```
netsh advfirewall firewall add rule name="ZQQ Block Inbound Connection" dir=in action=block remoteip=10.10.10.10
```
只允许某个来源的请求（防火墙默认是deny的）
```
netsh advfirewall firewall add rule name="ZQQ Allow Inbound Connection" dir=in action=allow remoteip=10.10.10.10
```
删除规则[Ref](https://serverfault.com/questions/1021182/windows-10-firewall-block-any-ip-address-but-one)
```
netsh advfirewall firewall del rule name="ZQQ Allow Inbound Connection"
```

## 用户
查看所有用户
```
net user
```
{{< hint info >}}
Administrator为管理计算机(域)的内置帐户
{{< /hint >}}
添加用户
```
net user <用户名> <密码> /add
```
修改密码
```
net user <用户名> <密码>
```
删除用户
```
net user <用户名> /delete
```
查看所有组
```
net localgroup
```
{{< hint info >}}
Administrators管理员组对计算机/域有不受限制的完全访问权
{{< /hint >}}
把用户`zqq`加入`administrators`组
```
net localgroup administrators zqq /add
```
查看组下用户
```
net localgroup administrators
```
查看用户`zqq`属于哪些组
```
net user zqq #如果该用户在Administrators组中，则该用户具有管理员权限
```
{{< hint warning >}}
**以上命令仅适用于普通账户不适用域账户**。普通账户和本地计算机绑定；域账户和域绑定，域下会有多台计算机。

他们之间的主要区别如下：
- 域账户在域控制器中进行身份验证，而普通账户在本地计算机中进行身份验证
- 域账户可以使用在域内的任何计算机上登录，而普通账户只能用于本地计算机登录
{{< /hint >}}