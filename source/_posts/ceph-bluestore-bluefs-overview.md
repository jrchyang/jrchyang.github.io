---
title: Ceph BlueStore BlueFS 代码详解之总体介绍
date: 2024-07-28 21:26:36
tags: [BlueStore,BlueFS]
categories: Ceph
description: BlueFS 总体介绍
---

## 1. 引言

BlueFS 是 BlueStore 的内部组件，通过 BlueRocksEnv 对接 RocksDB，用于处理 RocksDB 的文件相关的请求。其本身是一个用户态的日志型文件系统，所有元数据以 Log 的形式写到磁盘上，在启动时会读取 Log 文件重放所有操作，将全部元数据加载到内存内存中。本文及后续相关文件均已 Ceph v17.2.7 版本代码为基础进行分析。

本系列文章的重点是分析 BlueFS 的代码，不再介绍 BlueFS 出现的背景、优势等内容。

## 2. 分析步骤

我们大致按照如下流程分析 BlueFS 的代码：

1. 分析 BlueRocksEnv 以得到 BlueFS 对外提供的接口
2. 分析 BlueFS 的主要结构体，通过结构体分析 BlueFS 的主要结构
3. 分析 BlueFS 的初始化、读、写、元数据同步、加载、日志压实等几个主要处理逻辑

## 3. 展望

期望通过上述的分析过程，能够让大家对 BlueFS 有深入的理解。
