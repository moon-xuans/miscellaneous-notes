#### 1.重启docker服务后，容器自己停止了
解决办法在/etc/docker/daemon.json添加
`"live-resotre": true`

修改完成后，先执行`sudo systemctl daemon-reload`

之后，再重启docker后，容器不会停止

#### 2.删除容器时，报错`Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: 	Get http://%2Fvar%2Frun%2Fdocker.sock/v1.26/images/json: dial unix /var/run	/docker.sock: connect: permission denied“`

因为docker进程使用Unix Socket端口，不是TCP端口。而Unix socket属于root用户，因此需要root权限才能访问。

docker自身默认docker用户组具有读写Unix socket的权限，因此将当前用户添加进docker用户组即可。

```shell
sudo groupadd docker     #添加docker用户组
sudo gpasswd -a $USER docker     #将登陆用户加入到docker用户组中
newgrp docker     #更新用户组
docker ps    #测试docker命令是否可以使用sudo正常使用
```