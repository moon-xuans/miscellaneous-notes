#### 1.linux与windows主机ping的时候，主机可以ping通linux，但是linux无法ping通windows：

先看防火墙设置，是否有入栈规则，如果没有,则在cmd中执行命令，添加对应的规则
```cmd
New-NetFirewallRule -DisplayName "WSL" -Direction Inbound  -InterfaceAlias "vEthernet (WSL)"  -Action Allow
```
在linux上pingwindows上WSL对应的ip地址，ping通之后，则pingWLAN下的ip，如果不可以，则说明对应的icmp in被拒绝了，在防火墙对应的ICMP规则中，添加对应的网络(公用或者专用)



#### 2.WSL上暂时不支持systemctl

因此要使用service命令来操作,微软后期对wsl上的service进行了支持，无法使用
service的原因是systemd没有启用，因此可以在/etc/wsl.conf中进行配置，具体可看官方文档
https://learn.microsoft.com/zh-cn/windows/wsl/wsl-config

#### 3.切换WSL用户
需要在windows本机上看ubuntu的版本，路径为:C:\Users\86180\AppData\Local\Microsoft\WindowsApps

ubuntu版本是2004

然后使用命令，切换到root用户，也可以修改为自己
```cmd
ubuntu2004 config --default-user root

ubuntu2004 config --default-user moon
```
#### 4.在关闭hyper-v之后，wsl启动不了，显示0x80370102

首先，确保在控制面板中启动功能里面开启虚拟化之后，通过管理员身份打开cmd输入bcdedit
，查看hypervisorlaunchtype的状态，如果关闭了，则要将虚拟机监控程序设置为Auto
```cmd
bcdedit /set hypervisorlaunchtype Auto
```

#### 5.docker上mysql启动失败

wsl和windows上共用同一套端口，必须关闭windows上的mysql服务才可以。
