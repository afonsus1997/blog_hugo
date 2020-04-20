---
title: "mosquitto-mqtt-websocket"
date: 2018-08-07T10:43:30+08:00
categories: ["tech"] 
tags: ["conn"] 
draft: false
---

# 记一次 mosquito 安装支持 websockets

> 缘由：之前因为一个物联网项目，用到消息推送订阅，于是采取了 mqtt 协议，采用 mosquitto 作为中间件，但是由于 web 端需要实时接收消息，所以用 websockets 来解决，还好新的版本已经支持了，但是安装过程持续走低，所以记录一下。

## 安装依赖

```
# 如果没有编译工具
yum install gcc-c++
```

```
# mosquitto 依赖库
yum install openssl-devel
yum install c-ares-devel
sudo yum install libuuid-devel
```

```
# websockets 依赖库
(# 如果没有安装 git)
yum install git
git clone https://git.oschina.net/woniu201/libwebsockets.git
tar -zxvf libwebsockets-v1.5-stable.tar.gz
```

```
# 安装 cmake
yum install cmake
# 开始编译安装
cd libwebsockets-v1.5-stable
mkdir build
cd build
cmake ..
make 
make install
```

```
# 下载并安装 mosquitto 1.4.8
wget http://mosquitto.org/files/source/mosquitto-1.4.8.tar.gz
tar zxvf mosquitto-1.4.8.tar.gz
cd mosquitto-1.4.8
更改configure.mk中
WITH_WEBSOCKETS:=no
```

```
# 修改 mosquitto.conf 设置端口
```

![配置详情](https://github.com/Zhousantu/Zhousantu.github.io/blob/master/img/mosquitto_config.png?raw=true)

```
cd mosquitto-1.4.8
make
make install
# 把目录下的 .config 移动到 /etc/mosquitto
cp mosquitto.conf /etc/mosquitto
# 添加 user
adduser mosquitto
# 启动 mosquitto
mosquitto -c /etc/mosquitto/mosquitto.conf
结果出问题了！！！
​```
error while loading shared libraries: libwebsockets.so.4.0.0: cannot open shared object file: No such file or directory
​```
简单来说就是我们刚开始安装的依赖并没有派上用场（也就是没找到）
step1:locate libwebsockets.so # 查看依赖装哪儿去了
step2:/etc/ld.so.conf.d/ # 切换到该配置文件目录下
step3:vim libwebsockets.conf
写入：/usr/local/lib
step4:ldconfig
# 启动
mosquitto -c /etc/mosquitto/mosquitto.conf
```

#### 到此，完毕！