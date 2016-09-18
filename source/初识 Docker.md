title: 初识 Docker 
author: cwen
date:  2016-06-23
update:  2016-06-23
tags:
    - Docker 
    - golang 

---   
 
Docker 是一个开源的应用容器引擎，使用 golang 开发实现，让开发者可以打包他们的应用以及依赖包到一个可移植的容器中，然后发布到任何流行的 Linux 机器上，也可以实现虚拟化。容器是完全使用沙箱机制，相互之间不会有任何接口。      
Docker 项目的目标是实现轻量级的操作系统虚拟化解决方案。 Docker 的基础是 Linux 容器（LXC）等技术。   <!--more-->   
在 LXC 的基础上 Docker 进行了进一步的封装，让用户不需要去关心容器的管理，使得操作更为简便。用户操作 Docker 的容器就像操作一个快速轻量级的虚拟机一样简单

#### Docker 组成  
* Image - 镜像
* Containter - 容器
* Docker hub - 仓库    


#### 安装 Docker 
安装 Docker 要求 linux 内核版本不低于 3.13，Docker 依赖 linux 内核，使用 linux 的 namespae 实现进程的隔离，cgroup 来对资源的控制。(docker 与 linux 内核的关系以后单独研究，其实目前我不是太清楚，不敢瞎说) 如果你的内核版本低于 3.13，请自行 google 升级内核。       
  
查看自己内核版本信息 

``` 
$ uname -a
Linux Host 3.16.0-43-generic #58~14.04.1-Ubuntu SMP Mon Jun 22 10:21:20 UTC 2015 x86_64 x86_64 x86_64 GNU/Linux
```

或者    

``` 
$ cat /proc/version
Linux version 3.16.0-43-generic (buildd@brownie) (gcc version 4.8.2 (Ubuntu 4.8.2-19ubuntu1) ) #58~14.04.1-Ubuntu SMP Mon Jun 22 10:21:20 UTC 2015
```   
#### Ubuntu 安装   

###### 更新APT镜像源   
 
安装 apt-transport-https 包支持 https 协议的源   

```   
 $ sudo apt-get update 
 $ sudo apt-get install apt-transport-https ca-certificates
```

添加新的 gpg 密钥  

``` 
$ sudo apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys 58118E89F3A912897C070ADBF76221572C52609D   
```   
添加 Docker 的官方 apt 软件源   

```
$ sudo cat <<EOF > /etc/apt/sources.list.d/docker.list
deb https://apt.dockerproject.org/repo ubuntu-trusty main
EOF  
```   
> 非 trusty 版本的系统注意修改为自己对应的代号   
> deb https://apt.dockerproject.org/repo ubuntu-precise main      
> deb https://apt.dockerproject.org/repo ubuntu-trusty main      
> deb https://apt.dockerproject.org/repo ubuntu-wily main     
> deb https://apt.dockerproject.org/repo ubuntu-xenial main               


更新 apt 软件包缓存   

```
$ sudo apt-get update   
```  

如果系统中存在老版本的 Docker，请先删除  

```
$ sudo apt-get purge lxc-docker    
```  

检查 apt 源是否发生改变   

```  
$ apt-cache policy docker-engine   
```

###### 更新系统内核和安装可能需要的软件包      

```
$ sudo apt-get install linux-image-extra-$(uname -r)
```  
> `linux-image-extra` 允许你使用 `aufs` 文件系统    

###### 安装 Docker   

更新 apt 源

```
$ sudo apt-get update 
``` 

安装 Docker 

``` 
$ sudo apt-get install docker-engine     
```   

启动 Docker 守护进程 

``` 
$ sudo service docker start  
```  

检查 Docker 是否安装成功 

``` 
$ sudo docker run hello-world  
``` 


###### 使用脚本安装 Docker 

使用官方提供的安装脚本, 使用脚本我们可以直接运行，之前的步骤都可以省略    

```
$ sudo-i
$ wget -qO- https://get.docker.com/ | sh  
```

###### 把当前用户加入 Docker 用户组
运行 Docker 需 root 权限， 为了我们不必一直使用 root 权限，我们可以把用户加进 Docker 用户组 

``` 
$ sudo usermod -a -G docker [username]
``` 

> Ubuntu is all set up!



