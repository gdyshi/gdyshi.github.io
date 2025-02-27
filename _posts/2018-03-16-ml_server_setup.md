---
layout: post
title:  "机器学习服务器搭建"
categories: JavaScript
tags:  机器学习 TensorFlow ubuntu16.04 服务器搭建
---

* content
{:toc}

## 摘要
本文讲述如何搭建机器学习服务器及搭建步骤

## 引言


## 主题
### 安装依赖包
#### 安装系统依赖包
* 修改apt源

编辑`/etc/apt/sources.list`文件为以下内容
```
    deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
    deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
    ##测试版源
    deb http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
    # 源码
    deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted universe multiverse
    deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
    ##测试版源
    deb-src http://mirrors.aliyun.com/ubuntu/ xenial-proposed main restricted universe multiverse
    # Canonical 合作伙伴和附加
    deb http://archive.canonical.com/ubuntu/ xenial partner
    deb http://extras.ubuntu.com/ubuntu/ xenial main
```
执行`sudo apt-get update`

* 安装依赖文件
```
sudo apt-get install ssh nfs-common libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libhdf5-serial-dev protobuf-compiler --no-install-recommends libboost-all-dev libopenblas-dev liblapack-dev libatlas-base-dev libgflags-dev libgoogle-glog-dev liblmdb-dev git cmake build-essential
```

#### 安装python及依赖包
* 安装python`sudo apt-get install python3-pip python3-dev  python3-tk`
* 修改pip源

编辑~/.pip/pip.conf文件为以下内容
```
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
[install]
trusted-host=mirrors.aliyun.com
```
* 安装相关python包
```
sudo pip3 install --upgrade pip
sudo pip3 install --upgrade protobuf
sudo pip3 install --upgrade numpy
sudo pip3 install --upgrade matplotlib
sudo pip3 install --upgrade sklearn
sudo pip3 install --upgrade scipy ipython
```

### 安装显卡1080ti驱动
#### 方式1（手动）
* 删除原有驱动
```
sudo apt-get remove nvidia-*
sudo apt-get autoremove
sudo nvidia-uninstall
```

* 按组合键`Ctrl+Alt+F1`进入文本界面

* 停止依赖显卡的桌面服务`sudo service lightdm stop`
* 安装驱动
`sudo ./NVIDIA-Linux-x86_64-3xx.xx.run -no-x-check -no-nouveau-check -no-opengl-files`
> NVIDIA-Linux-x86_64-3xx.xx.run 为从NVIDIA官网下载的最新驱动

* 重启桌面服务`sudo service lightdm restart`



#### 方式2（apt-get）
* 添加源
`sudo add-apt-repository ppa:graphics-drivers/ppa`
* 下载新源中的包名
`sudo apt-get update`
* 安装驱动
```
sudo apt-get install nvidia-38*
sudo apt-get install mesa-common-dev
sudo apt-get install freeglut3-dev
```

### 安装CUDA
* 执行安装命令`./cuda_8.0.61_375.26_linux-run`
> 注意：安装过程中不要安装cuda自带驱动

* 添加环境变量

在`/etc/profile`文件追加`export PATH=/usr/local/cuda-8.0/bin:$PATH`
在`/etc/ld.so.conf`文件追加`/usr/local/cuda-8.0/lib64`
执行命令,使环境变量生效
```
source /etc/profile
sudo ldconfig
```


### 安装CUDNN
* 执行安装命令`sudo dpkg -i libcudnnxxx-1+cudax.x_amd64.deb`

### 安装TensorFlow
```
sudo pip3 install --upgrade tensorflow-gpu
```

## 附录


## 参考
---
- [TensorFlow官方安装教程](https://www.tensorflow.org/install/install_linux)
- [Ubuntu16.04换源](https://blog.csdn.net/happywho250/article/details/52506321)
- [pip换源](https://www.cnblogs.com/microman/p/6107879.html)
- [CUDA下载链接](https://developer.nvidia.com/cuda-downloads)
- [CUDA官方安装教程](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/#axzz4VZnqTJ2A)
- [CUDNN下载链接](https://developer.nvidia.com/cudnn)
