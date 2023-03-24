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
powershell
```

## 网络
