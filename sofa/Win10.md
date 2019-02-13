# Win10

## 快捷键

Function|ShortCut|
---|---
create virtual window |   Ctrl + Win + D
switch virtual window |   Win + Tab


## cmd

对标记内容复制: Ctrl + Insert

## win原生软件
|Sofa|Win + R + ?|
|---|:---:|
画板 |  mspaint
注册表 | regedit
计算器 | calc

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
