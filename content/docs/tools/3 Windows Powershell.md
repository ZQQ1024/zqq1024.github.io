---
title: "Windows PowerShell"
weight: 3
bookToc: false
---

# Windows PowerShell

主要介绍`Windows PowerShell`常用命令

## 文件
切换目录
```
Set-Location C:\Users\zqq\Desktop # 或cd C:\Users\zqq\Desktop
```
当前目录
```
Get-Location # 或pwd
```
当前目录中的文件
```
Get-ChildItem # 或ls
```
{{< hint info >}}
**PowerShell中看似常规的`ls`等命令都是xxx-xxx风格命令的别名**
```
help ls

名称
    Get-ChildItem

语法
    Get-ChildItem [[-Path] <string[]>] [[-Filter] <string>]  [<CommonParameters>]

    Get-ChildItem [[-Filter] <string>]  [<CommonParameters>]

别名
    gci
    ls
    dir
```
{{< /hint >}}

查看`1.txt`文件内容
```
Get-Content 1.txt # 或cat 1.txt，如果文件太长，可以 more 1.txt
```
删除`1.txt`文件
```
Remove-Item 1.txt # 或rm 1.txt，可以删除文件夹
```
基于文件名`1.txt`在C盘中查找文件，并忽略无访问权限的报错
```
Get-ChildItem -Path "C:\" -Recurse -Filter "1.txt" -ErrorAction SilentlyContinue -ErrorVariable AccessDenied
```
基于文件内容`123`查找文件，不包含子目录
```
Select-String -Path "C:\Users\zqq\Desktop\*.txt" -Pattern "123"
```
基于文件内容`123`查找文件，包含子目录
```
Get-ChildItem -Path "C:\Users\zqq" -Recurse -Include "*.txt","*.log","*.csv" | ForEach-Object { If(Select-String -Path $_.FullName -Pattern "123") { $_.FullName } }
```

## 进程

全部进程
```
Get-Process
```
基于进程名`cmd`查找进程
```
Get-Process -Name cmd
```
基于`pid`关闭进程
```
Stop-Process -id 10760
```
取消运行`ps1`脚本限制
```
Set-ExecutionPolicy RemoteSigned
```
{{< hint info >}}
在默认情况下，`PowerShell`防止通过执行未受信任的脚本（如未签名的脚本等）对系统造成潜在的安全威胁。  
其他选项包含：
- `Restricted`: 完全阻止脚本执行
- `AllSigned`: 所有脚本都是受信任的
- `RemoteSigned`: 在远程计算机或从Internet上获取的脚本需要签名，而本地计算机上的脚本不需要
{{< /hint >}}
运行`1.ps1`脚本文件
```
cd C:\Users\zqq\Desktop
.\1.ps1
```
获取所有特性（可以通过`-FeatureName`指定特定Feature）
```
Get-WindowsOptionalFeature -Online
```
启用telnet client
```
Enable-WindowsOptionalFeature -Online -FeatureName TelnetClient
```
启用rdp（`fDenyTSConnections`默认为1拒绝改为0，修改不用重启）
```
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections" -Value 0
```
禁用rdp
```
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections" -Value 1
```
获取`fDenyTSConnections`配置
```
Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -name "fDenyTSConnections"
```
PowerShell访问CMD
```
cmd
```

## 网络
访问`百度`网页，并打印返回的HTML
```
$content = Invoke-WebRequest -Uri "https://www.baidu.com"
$content.Content
```
下载文件
```
Invoke-WebRequest -Uri "https://example.com/file.zip" -OutFile "C:\path\to\save\file.zip"
```
检测哪个进程打开了`3389`端口
```
Get-Process -Id (Get-NetTCPConnection -LocalPort 3389).OwningProcess
```
查询防火墙状态
```
Get-NetFirewallProfile | Select-Object name, enabled
```
关闭防火墙
```
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False
```
打开防火墙
```
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled True
```
查询防火墙规则
```
Get-NetFirewallRule -Name Microsoft-Windows-Unified-Telemetry-Client
```
拒绝`10.10.10.10`来源的请求
```
New-NetFirewallRule -DisplayName "My Custom Firewall Rule" -Direction Inbound -LocalPort 80 -Protocol TCP -RemoteAddress 10.10.10.10 -Action Block
```
删除规则
```
Remove-NetFirewallRule -DisplayName "My Custom Firewall Rule"
```

## 用户
查看所有本地用户
```
Get-LocalUser
```
添加用户，密码为`123456`
```
New-LocalUser -Name "test" -Password (ConvertTo-SecureString "123456" -AsPlainText -Force)
```
修改密码密码为`789000`
```
Set-LocalUser -Name "test" -Password (ConvertTo-SecureString "789000" -AsPlainText -Force)
```
删除用户`test`
```
Remove-LocalUser -Name "test"
```
查看所有组
```
Get-LocalGroup
```
把用户`test`加入`administrators`组
```
Add-LocalGroupMember -Group "Administrators" -Member "test"
```
查看组`administrators`下的用户
```
Get-LocalGroupMember -Group "administrators"
```