# Win10

## 快捷键

Function|ShortCut|
---|---
create virtual window |   Ctrl + Win + D
switch virtual window |   Win + Tab


## cmd

对标记内容复制: Ctrl + Insert

## 强制删除文件

```text
1. 管理员运行 cmd
2. 输入命令行:
    强制删除文件: del/f/s/q 盘符:\文件名  （强制删除文件，文件名必须加文件后缀名）
    强制删除文件夹: rd/s/q [drive:]path  （强制删除文件文件夹和文件夹内所有文件）
```

## win原生软件
|Sofa|Win + R + ?|
|---|:---:|
画板 |  mspaint
注册表 | regedit
计算器 | calc
系统属性 | sysdm.cpl
服务 | services.msc

<br />
## 小功能

### powershell -> cmd

>注册表管理器\计算机\HKEY_CLASSES_ROOT\Directory\Background\shell\Powershell  
>修改权限，设置当前账号完全控制, Powershell的ShowBasedOnVelocityId的重命名HideBasedOnVelocityId  
>这样就关闭了Ctrl + 右键显示powershell  
>同理Ctrl + 右键显示cmd在cmd项上的HideBasedOnVelocityId重命名为ShowBasedOnVelocityId即可  

<br />

### win软件图标删除左下角的箭头

> cmd管理员运行
> /k reg delete "HKEY_CLASSES_ROOT\lnkfile" /v IsShortcut /f & taskkill /f /im explorer.exe & start explorer.exe

<br />

### 网络修改

> cmd管理员运行
> netsh interface ip set address "以太网" static 192.168.0.2 255.255.255.0 192.168.0.1

<br />

### 查看端口占用

> cmd管理员运行
> netstat -ano|findstr "8080" 查看PID即最后一列, 在任务管理器中删除该程序
> taskkill /pid processid -f (比如: taskkill /pid 82789 -f, 强制停止 pid = 82789 的进程)

> cmd管理员运行
> taskkill /f /t /im "app.exe" (比如: taskkill /f /t /im "cmd.exe", 强制终止 cmd.exe 程序, 一般终止保护程序)
