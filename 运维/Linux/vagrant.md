#### 1.vagrant up 卡在booting vm(不确定)

这是之前使用wsl的时候，将hypervisor设置成了Auto，需要关闭
```cmd
bcdedit /set hypervisorlaunchtype off
```

#### 2.vagrant up 卡在ssh private key

使用的virtualbox以及vagrant版本较老，到官网按照最新版本即可。

#### 3.配置虚拟机网络时，没有找到virtualbox的ip

在设备管理器的网络适配器中查看发现存在virtualbox的虚拟网卡，直接在virtualbox
中的工具中的属性设置网卡的ip即可。


test
