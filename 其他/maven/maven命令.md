# maven命令

## 1.常用命令

### 1.1.compile

编译源代码

### 1.2.test

运行测试

### 1.3.clean

清理maven项目

### 1.4.package

完成了项目编译、单元测试、打包功能，但是并没有把打好的可执行jar包(或者其他形式的包)部署到本地maven仓库或者远程私服。

### 1.5.install

完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包(或者其他形式的包)部署到本地maven仓库，但没有部署到远程私服。

### 1.6.deploy

完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包(或者其他形式的包)部署到本地maven仓库和远程maven私服仓库。

## 2.其他命令

### 2.1.删除由于网络波动下载失败的包

```shell
find /home/axuan/programme/maven-repo -name "*.lastUpdated" -exec grep -q "Could not transfer" {} ; -print -exec rm {} ;
```

### 2.2.查看maven中生效的配置文件

```shell
mvn help:effective-settings  
```

### 2.3.查看maven配置文件的选择过程

```shell
mvn -X 
```

## 3.其他

配置的时候注意名称一定要是settings.xml，否则找不到会生成一个新的配置文件。