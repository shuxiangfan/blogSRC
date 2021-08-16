---
title: 在ManjaroLinux上编译Android源码
date: 2021-08-15 13:43:17
tags: Linux
---

# 在ManjaroLinux上编译Android源码

第一次写教程，语文不好大佬手下留情哈哈哈哈

**刷机有风险！请备份好重要数据！自行承担风险！**

## 系统要求

- Manjaro Linux 21.0.7及以上，运行在x86处理器上

- 至少4GB RAM和300GB磁盘空间

- 畅通的互联网连接

- 基本的linux知识

- 脑子和手


## 配置软件源

### 更换软件源

刚安装完的Manjaro需要更换镜像源 

```
sudo pacman-mirrors -i -c China -m rank
```

稍等片刻，选择自己需要的镜像源

#### 进行全面系统更新

```
sudo pacman -Syu
```

### 安装依赖

#### 安装yay

```
sudo pacman -S yay
```

#### 安装构建依赖

```
yay -S lineageos-devel
```
根据需要选择依赖（一路回车和输密码）

至此，构建依赖安装完成

#### 建立交换文件

低内存（16G以下）用户请看

使用dd去创建一个由你自己指定大小的交换文件。例如，创建一个 20 GiB 的交换文件:

```
sudo dd if=/dev/zero of=/swapfile bs=1G count=20 status=progress
```

为交换文件设置权限（交换文件全局可读是一个巨大的本地漏洞）：

```
sudo chmod 600 /swapfile
```

创建正确大小的文件后，将其格式化用来作为交换文件：

```
sudo mkswap /swapfile
```

启用交换文件：

```
sudo swapon /swapfile
```

最后，编辑 /etc/fstab，在为交换文件添加一个条目：
```
/swapfile none swap defaults 0 0
```

## 下载系统源代码

### 配置git

设置git的用户名

```
git config --global user.name "你的用户名"
```
设置邮箱

```
git config --global user.email "你的邮箱"
```
由于国内网络环境特殊，可以设置从镜像站下载

```
git config --global url.https://mirrors.bfsu.edu.cn/git/AOSP/.insteadof https://android.googlesource.com
git config --global url.https://github.com.cnpmjs.org/.insteadof https://github.com
```

### 指定repo更新地址

repo的运行过程中会尝试访问官方的git源更新自己，如果想使用bfsu的镜像源进行更新，可以将如下内容复制到你的~/.bashrc里

```
export REPO_URL='https://mirrors.bfsu.edu.cn/git/git-repo'
```
然后运行

```
source ~/.bashrc
```

## 建立工作区

在符合系统要求的磁盘上建立工作文件夹

```
mkdir android
cd android
```

### 初始化仓库

Repo 可以在必要时整合多个 Git 代码库

Repo是一种对 Git 构成补充的 Google 代码库管理工具

到Github或其它社区找到您喜欢的ROM中名为platform_manfest或android的git仓库，并找到他们提供的命令。

例如：

#LineageOS_18.1
```
repo init -u git://github.com/LineageOS/android.git -b lineage-18.1
```
#exTHmUI-11
```
repo init -u https://github.com/exthmui/android.git -b exthm-11
```
Tip:如果你只想编译，加上--depth=1参数可以节省磁盘空间

## 同步仓库

```
repo sync
```
这将会从网络上下载源码，这个过程可能需要耗费一点时间，请耐心等待

出现问题可以多同步几遍

## 编译

这里使用第三方ROM官方支持的手机进行演示，checkin教学将在第二部进行（挖坑）

### 环境准备

```
source build/envsetup.sh
```
```
lunch {ROM名称}_{手机开发代号}-{构建类型}
```

例如：
```
lunchc aosp_maple-eng
```

将花括号去掉，在对应的位置上填上您需要的值

这将会自动下载设备树、内核源码和供应商私有部分

### 低内存解决方案

Android11：在源码根目录执行

```
cd build/soong && git fetch https://github.com/masemoel/build_soong_legion-r 11 && git cherry-pick b45c5ae22f74f1bdbb9bfbdd06ecf7a25033c78b && git cherry-pick e020f2130224fbdbec1f83e3adfd06a9764cca87 && cd ../..
```
这将会从远程拉取修补补丁来限定java使用的内存

### 正式编译

```
mka bacon
```
#或者
```
mka otapackage
```
这会花费几个小时甚至数十个小时（取决于系统性能）

### 编译完成

如果没有错误，会出现类似build completed successfully(绿色字体)的输出，这时可以到out/target/product/{手机代号}/目录下找到刷机包

部分ROM还会在编译完成告知用户刷机包文件位置和文件名

## 刷入

现在您得到了刷机包，您可以在自定义Recovery中刷入您的刷机包了！

如果不开机…….那我也没办法（逃）

## 引用

https://mirrors.bfsu.edu.cn/help/archlinuxcn/

https://mirrors.bfsu.edu.cn/help/git-repo/

https://github.com/LineageOS/android

https://github.com/exthmui/android/tree/exthm-11

https://source.android.google.cn/setup/build/building?hl=zh-cn

https://source.android.google.cn/setup/develop?hl=zh-cn

https://wiki.archlinux.org/title/Swap_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)