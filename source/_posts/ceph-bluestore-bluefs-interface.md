---
title: Ceph BlueStore BlueFS 代码详解之接口说明
date: 2024-07-28 21:26:36
tags: [BlueStore,BlueFS]
categories: Ceph
description: 梳理 BlueFS 对外提供的接口
---

## 1. 说明

本文将从 BlueRocksEnv 入手，分析 BlueFS 对外提供的几个主要的接口，一些简单的接口（如重命名、判断是否存在、查询状态、加锁等）不做详细分析，读者可自行理解。

## 2. BlueRocksEnv

RocksDB 通过 EnvWrapper 封装了其调用的文件相关操作的接口。BlueRocksEnv 继承了该类，内部通过调用 BlueFS 相关的接口实现相关的功能，我们通过分析 BlueRocksEnv 的实现可得出如下 BlueFS 提供的接口。

基本上对于文件的所有操作都是通过文件句柄执行的，BlueRocksEnv 中共有如下三个获取文件句柄的接口：

```C++
// 获取一个顺序读文件句柄
rocksdb::Status NewSequentialFile(
  const std::string& fname,
  std::unique_ptr<rocksdb::SequentialFile>* result,
  const rocksdb::EnvOptions& options) override;

// 获取一个随机读文件句柄
rocksdb::Status NewRandomAccessFile(
  const std::string& fname,
  std::unique_ptr<rocksdb::RandomAccessFile>* result,
  const rocksdb::EnvOptions& options) override;

// 获取一个写文件句柄
rocksdb::Status NewWritableFile(
  const std::string& fname,
  std::unique_ptr<rocksdb::WritableFile>* result,
  const rocksdb::EnvOptions& options) override;
```

通过以上接口实现的代码分析，其最终调用的 BlueFS 的接口如下：

```C++
int BlueFS::open_for_read(
  std::string_view dirname,
  std::string_view filename,
  FileReader **h,
  bool random)

int BlueFS::open_for_write(
  std::string_view dirname,
  std::string_view filename,
  FileWriter **h,
  bool overwrite)
```

其中无论是获取顺序读文件句柄还是获取随机读文件句柄，调用的都是 `open_for_read` 接口，区别在于 `random` 参数。如果 `random` 为真，表明是随机读，则设置预读大小为 4K；如果为假，表明是顺序读，则设置预读大小为 `bluefs_max_prefetch` （默认是 1M）。以上两个接口最终会返回 `FileReader` 和 `FileWriter` 对象，其中读句柄会封装到 `BlueRocksSequentialFile` 和 `BlueRocksRandomAccessFile` 中，分别对应顺序读和随机读，写句柄会封装到 `BlueRocksWritableFile` 中，RocksDB 通过这三类对象来操作文件，那么文件的读、写等主要接口即在这个类中。

`BlueRocksSequentialFile` 类中主要调用 BlueFS 的接口如下：

```C++
int64_t read(FileReader *h, uint64_t offset, size_t len,
             ceph::buffer::list *outbl, char *out) {
  return _read(h, offset, len, outbl, out);
}
```

`BlueRocksRandomAccessFile` 类中主要调用 BlueFS 的接口如下：

```C++
int64_t read_random(FileReader *h, uint64_t offset, size_t len, char *out) {
  return _read_random(h, offset, len, out);
}
```

`BlueRocksWritableFile` 类中主要调用 BlueFS 的接口如下：

```C++
// 追加数据，BlueFS 为仅追加文件系统，不支持随机写
void BlueFS::append_try_flush(FileWriter *h, const char* buf, size_t len)
// 同步指定文件的数据
void BlueFS::flush(FileWriter *h, bool force)
// 同步指定文件的元数据
int BlueFS::fsync(FileWriter *h)
```

另外再 BlueRocksEnv 调用的部分接口中有会导致元数据变化的，这些接口都会调用同步整个文件系统元数据的接口，如下：

```C++
void BlueFS::sync_metadata(bool avoid_compact)
```

后续我们会从这些基本的接口出发，详细分析 BlueFS 运行逻辑。
