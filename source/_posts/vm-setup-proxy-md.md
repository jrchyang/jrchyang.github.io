---
title: 虚拟机设置代理
date: 2023-04-23 13:00:39
tags: 配置代理
categories: 配置工具
description: 如何在虚拟机中设置代理以共享宿主机的代理工具
---

## 1. 场景

一些日常开发、调试工作一般在虚拟机中进行，如需从 GitHub 下载源码，会由于网络问题导致经常下载失败，一般可以通过配置 ssh 或直接下载 zip 代码包来解决。但对于一些较大型的项目，会包含一些子模块并需要安装很多其他依赖项，如继续使用上述方式将导致将导致整个过程非常繁琐且不一定能够解决，可以通过终端配置代理的方式来解决，具体方法如下文。

## 2. 配置代理

### 2.1. 查看宿主机 IP

```Shell
# 在windows平台下，通过 ipconfig 命令查看
ipconfig
```

### 2.2. 查看代理软件端口

这里直接查看自己使用的代理软件即可，以 clash 为例，端口为 7890。另外，要打开 **Allow LAN** 选项。

### 2.3. 虚拟机配置

在终端执行如下命令：

```Shell
# you_host_ip 为宿主机 ip
ip=$you_host_ip
export https_proxy=http://$ip:7890;export http_proxy=http://$ip:7890;export all_proxy=socks5://$ip:7890
```

可执行如下命令检查是否正常：

```Shell
curl https://www.google.com
```

### 2.4. 脚本

为方便起见可以将上述操作封装到如下脚本中：

```Shell
#!/bin/bash

function usage()
{
  echo "$0 unset"
  echo "$0 set \$ip"
  exit
}

function set_proxy()
{
  if [ -z "$ip" ];then
    usage
  fi

  export https_proxy=http://$ip:7890
  export http_proxy=http://$ip:7890
  export all_proxy=socks5://$ip:7890
}

function unset_proxy()
{
  unset https_proxy
  unset http_proxy
  unset all_proxy
}

op=$1
ip=$2

if [ "$op" = "set" ];then
  set_proxy
elif [ "$op" = "unset" ];then
  unset_proxy
else
  usage
fi
```

在终端执行如下命令进行设置和取消：

```Shell
# 设置代理
. ./proxy.sh set $ip
# 取消代理
. ./proxy.sh unset
```
