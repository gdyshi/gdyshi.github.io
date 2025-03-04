---
layout: post
title:  "机器学习服务器搭建"
categories: 环境搭建
tags:  机器学习 TensorFlow ubuntu16.04 服务器搭建
---

* content
{:toc}

## 摘要
> 本文讲述如何搭建机器学习服务器及搭建步骤
---

## 引言


## 主题
### 安装依赖包
#### 安装系统依赖包
```
sudo apt-get install ssh nfs-common libprotobuf-dev libleveldb-dev libsnappy-dev libopencv-dev libhdf5-serial-dev protobuf-compiler --no-install-recommends libboost-all-dev libopenblas-dev liblapack-dev libatlas-base-dev libgflags-dev libgoogle-glog-dev liblmdb-dev git cmake build-essential
```

#### 安装python及依赖包
```
sudo apt-get install python3-pip python3-dev  python3-tk
# 修改pip源 ~/.pip/pip.conf
sudo pip3 install --upgrade pip
sudo pip3 install --upgrade protobuf
sudo pip3 install --upgrade numpy
sudo pip3 install --upgrade matplotlib
sudo pip3 install --upgrade sklearn
sudo pip3 install --upgrade scipy ipython
```

### 安装显卡1080ti驱动
- 方式1（手动）

1. 删除原有驱动
```
sudo apt-get remove nvidia-*
sudo apt-get autoremove
sudo nvidia-uninstall
```

2. 停止依赖显卡的桌面服务

按组合键“Ctrl+Alt+F1”进入文本界面
```
# 停止桌面服务
sudo service lightdm stop
# 安装驱动
sudo ./NVIDIA-Linux-x86_64-3xx.xx.run -no-x-check -no-nouveau-check -no-opengl-files
# 重启桌面服务
sudo service lightdm restart
```
> NVIDIA-Linux-x86_64-3xx.xx.run 为从NVIDIA官网下载的最新驱动

- 方式2（apt-get）
```
# 添加源
sudo add-apt-repository ppa:graphics-drivers/ppa
# 下载新源中的包名
sudo apt-get update
# 安装驱动
sudo apt-get install nvidia-38*
sudo apt-get install mesa-common-dev
sudo apt-get install freeglut3-dev
```

### 安装CUDA
```
# 执行安装命令
./cuda_8.0.61_375.26_linux-run # 安装过程中不要安装cuda自带驱动
# 添加环境变量
sudo echo "export PATH=/usr/local/cuda-8.0/bin:$PATH" >> /etc/profile
sudo echo "/usr/local/cuda-8.0/lib64" >> /etc/ld.so.conf
```
> [CUDA下载链接](http://developer.nvidia.com/cuda-downloads)

### 安装CUDNN
```
sudo dpkg -i libcudnnxxx-1+cudax.x_amd64.deb
```
> [CUDNN下载链接](https://developer.nvidia.com/cudnn)

### 安装TensorFlow
```
sudo pip3 install --upgrade tensorflow-gpu
```

## 附录


## 参考
---
- [TensorFlow官方安装教程](https://www.tensorflow.org/install/install_linux)
- [Ubuntu16.04换源](http://blog.csdn.net/happywho250/article/details/52506321)
- [pip换源](https://www.cnblogs.com/microman/p/6107879.html)
- [CUDA官方安装教程](http://docs.nvidia.com/cuda/cuda-installation-guide-linux/#axzz4VZnqTJ2A)
