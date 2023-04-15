# 340实验室GPU服务器的LXD虚拟化

实验室有了很多台服务器，因为实验人数比较多，为了减轻配置服务器的负担，且防止小白会运行损害系统的命令

![image](https://github.com/Chaos-Hu-edu/340_GPU_SERVER/blob/main/img/img1.gif)

所以使用lxd做了虚拟化。

参考链接：

[实验室GPU服务器的LXD虚拟化](https://github.com/a15720934530/LXD_GPU_SERVER)

[lxd官方文档](https://linuxcontainers.org/lxd/docs/master/)

## [用户手册](https://github.com/Chaos-Hu-edu/340_GPU_SERVER/blob/main/lxd容器使用说明/普通用户使用.md)

## [管理员手册](https://github.com/Chaos-Hu-edu/340_GPU_SERVER/blob/main/lxd容器使用说明/管理员.md)

## 1. 第一步：宿主机的安装与配置

### 1.1. 服务器系统的安装

安装Ubuntu系统

最好安装 ubuntu 18.04版本

并且把自动更新关闭

### 1.2. 服务器显卡驱动的安装

尽量安装最新的驱动

[显卡驱动安装](https://medium.com/@cjanze/how-to-install-tensorflow-with-gpu-support-on-ubuntu-18-04-lts-with-cuda-10-nvidia-gpu-312a693744b5)

30系显卡智能安装cuda11以上的驱动，不需要安装CUDA，cuDNN，可以直接通过[pytorch官网](https://pytorch.org/)进行安装对于版本的CUDA和cuDNN

## 2. 第二步：LXD的安装与初始化

### 2.1. 安装LXD

LXD实现虚拟容器

ZFS用于管理物理磁盘，支持LXD高级功能

bridge-utils用于搭建网桥

需要先联网更新系统库

`apt-get upgrade`

安装LXD，ZFS和bridge-utils

`sudo apt-get install lxd zfsutils-linux bridge-utils`

### 2.2. 配置网桥

因为学校服务器室只开发了一个端口，所以所有容器都使用跳板机进行连接。

### 2.3. 配置ZFS

首先，运行sudo fdisk -l列出服务器上的可用磁盘和分区，服务器一般有两个盘，固态为系统盘，机械硬盘为数据盘，现在使用机械盘作为容器的存储卷

#### 2.3.1. 查看分区（可选）

查看所有分区信息

`sudo fdisk -l`

进行分区

`sudo fdisk /dev/sdb`

#### 2.3.2. 创建块设备

在块设备/dev/sdb上创建一个ZFS存储池local

`sudo lxc storage create local zfs source=/dev/sdb`

注意，创建存储池后，需要修改宿主机中的硬盘挂载

`/etc/fstab`

### 2.4. LXD初始化

`sudo lxd init`

#### 2.4.1. 再次配置

`sudo lxc profile edit default`

## 3. 第三步：容器的创建

### 3.1. 加速源

使用清华的镜像源

`sudo lxc remote add tuna-images https://mirrors.tuna.tsinghua.edu.cn/lxc-images/ --protocol=simplestreams --public`

列出可用的镜像

`sudo lxc image list tuna-images:`

### 3.2. 创建ubuntu容器

使用清华源中的ubuntu镜像创建一个叫test的容器

`sudo lxc launch tuna-images:ubuntu/18.04 test` 

### 3.3. 进入容器

`sudo lxc exec test bash`

登录的是root用户，在这个容器中已经存在了一个叫ubuntu的用户

### 3.4. 修改密码

`passwd root`

`passwd ubuntu`

容器里的ubuntu是一个很精简的系统，需要安装各种软件

### 3.5. 安装ssh

`apt install ssh`

在这之前可能需要更新源并更换源

### 3.6. 通过ssh连接容器

查看容器与宿主机的IP

退出容器

`exit`

在宿主机查看容器

`sudo lxc list`

查看宿主机IP地址

`ip addr`

端口转发

`sudo lxc config device add 容器name proxy0 proxy listen=tcp:宿主机IP地址:端口 connect=tcp:容器IP:22 bind=host`

## 4. 第四步：初始容器的配置

使用ssh连接容器并配置

`ssh ubuntu@宿主机IP -p 端口`

### 4.1. 更换源

备份原来的源

`sudo mv /etc/apt/sources.list  /etc/apt/sources.list.bak`

编辑写入网易源

`sudo vim /etc/apt/sources.list`

```
deb http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.163.com/ubuntu/ bionic-backports main restricted universe multiverse
```

### 4.2. 为容器添加显卡

回到宿主机

为容器添加所有GPU：

`lxc config device add 容器name gpu gpu`

添加指定GPU：

`lxc config device add 容器name gpu0 gpu id=0` 

安装驱动

添加好显卡后，就相当于给容器安装了显卡，回到容器，然后安装显卡驱动

显卡驱动与宿主机的显卡版本必须一致，只需要安装显卡驱动。

安装时不需要安装到内核

`sudo sh ./NVIDIA-Linux-X86_64-[YOURVERSION].run --no-kernel-module`

### 4.3. 安装图形化界面

刷新源

`sudo apt update`

安装无推荐软件的ubuntu桌面

`sudo apt install --no-install-recommends ubuntu-desktop`

### 4.4. 安装远程连接

使用安装脚本

`sudo apt install git`

`git clone https://github.com/shenuiuin/LXD_GPU_SERVER`

打开文件夹

`cd LXD_GPU_SERVER/`

赋予脚本可执行权限

`sudo chmod a+x xrdp-installer-1.2.sh`

脚本会下载一些文件，需要有Downloads文件夹

`mkdir -p ~/Downloads`

安装脚本

`./xrdp-installer-1.2.sh -c -l -s`

[脚本源地址](http://c-nergy.be/blog/?cat=79)

其他桌面的需求

[kde桌面环境以及xrdp安装](https://www.hiroom2.com/2018/05/07/ubuntu-1804-xrdp-kde-en/)
[xfce桌面环境以及xrdp安装](https://www.hiroom2.com/2018/05/07/ubuntu-1804-xrdp-xfce-en/)
[xrdp解决声音重定向](http://c-nergy.be/blog/?p=12469)

### 4.5. 远程连接测试（以后再做）

端口转发

在安装好XRDP后，与之前一样，使用宿主机的端口进行转发

`sudo lxc config device add 容器name proxy0 proxy listen=tcp:宿主机IP地址:端口 connect=tcp:容器IP:3389 bind=host`

可以通过windows的远程连接来使用容器（windows运行mstsc）

### 4.6.  安装需要的软件

显示linux系统信息

`sudo apt install neofetch`

`neofetch`

查看CPU运行以及内存占用情况

`sudo apt install htop`

`htop`

查看显卡运行情况

`nvidia-smi`

实时查看显卡运行情况

`watch -n 0.1 nvidia-smi`

## 5. 第五步：容器管理

查看zfs存储卷的占用情况

`zpoll list`

为容器修改参数配置

[容器参数配置说明](https://linuxcontainers.org/lxd/docs/master/containers)

配置容器参数

`lxc config edit YourContainerName`

一般使用以下的配置即可满足

![my-logo.png](https://github.com/Chaos-Hu-edu/340_GPU_SERVER/blob/main/img/img2.gif)

配置默认容器参数

`sudo lxc profile edit default`

## 6. 第六步：容器模板

把配置好的容器作为模板，保存为镜像

停止容器

`sudo lxc stop 容器名字`

将容器保存为ubuntudemo镜像

`sudo lxc publish 容器名字 --alias ubuntudemo --public`

修改镜像的描述

`lxc image edit <image_ID>`

```python
properties:
  description: xxx
  os: Ubuntu
  release: 18.4
```

从模板镜像中新建容器，容器创建好后修改它的配置文件：添加端口映射（远程连接与SSH）、添加显卡（显卡驱动已经有了）、配置硬件参数

`sudo lxc image list`

`sudo lxc launch FINGERPRINT name --target=location`

 





## 番外

### 在lxd容器中使用docker

`lxc config edit YourContainerName`

在config中添加

![my-logo.png](https://github.com/Chaos-Hu-edu/340_GPU_SERVER/blob/main/img/img3.gif)

然后容器重启

`lxc restart YourContainerName`

[在容器内安装docker](https://docs.docker.com/install/linux/docker-ce/ubuntu/)

共享目录

path1为宿主机路径，path2为容器内路径

`lxc config set yourContainerName security.privileged true`

`lxc config device add privilegedContainerName shareName disk source=path1 path=path2`

若容器内对共享目录没有权限，只需将宿主机目录路径权限给足

`sudo chmod -R 777 path1`





 











