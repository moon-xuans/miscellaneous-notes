linux与windows主机ping的时候，主机可以ping通linux，但是linux无法ping通windows：

1.先看防火墙设置，是否有入栈规则，如果没有,则在cmd中执行命令，添加对应的规则
```cmd
New-NetFirewallRule -DisplayName "WSL" -Direction Inbound  -InterfaceAlias "vEthernet (WSL)"  -Action Allow
```
在linux上pingwindows上WSL对应的ip地址，ping通之后，则pingWLAN下的ip，如果不可以，则说明对应的icmp in被拒绝了，在防火墙对应的ICMP规则中，添加对应的网络(公用或者专用)