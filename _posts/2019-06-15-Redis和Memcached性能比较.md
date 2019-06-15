---
layout: post
title: Redis和Memcached性能比较
categories: 分布式
description: Redis VS Memcached
keywords: Redis,Memcached
---

## Redis VS Memcached

### 背景介绍：

项目中需要保存用户的session信息，保证“一次登陆，多处共享”，每次重新登陆时，需要先删除掉session，然后再进行保存。

每个用户有40条左右的信息需要保存，每条信息的大小在10k以上，所以每个用户有400k以上的数据需要保存。

### 方案选择：

最开始公司的方案是采用`memcached`进行缓存，一个key对应一个value，也就是说每个用户的数据需要保存40次，删除也是一样，根据每个用户不同的key去删除。

为了提升性能，技术团队决定采用`redis`的hash结构来保存，一个用户只需要保存一次，只不过一个hash的数据大小为400k以上。

`memcached`采用`XMemcached`客户端，高并发下可以多线程操作；`redis`采用`Jedis`客户端。

### 问题描述：

项目上线后，某次高峰时段，500多QPS，redis服务器瘫痪（redis有集群），导致用户不能正常登陆，业务量骤降。

网上普遍的认识是redis服务器10k以内的数据大小性能都特别高，10k开始性能下降，当数据量更大，到100k左右时，redis的性能比memcached要差。

但是这种结论只是服务器的性能比较，在项目中需要结合客户端的操作才能对服务器进行操作，所有客户端的性能问题也很重要。

于是我尝试进行了下面的实验。。。

### 环境准备：

所有的服务都是在本地搭建的，系统使用Windows子系统（WLS），`redis`和`memcached`服务安装在这个系统上：

```properties
OS = ubuntu 18.04
redis = 4.0.9
memcached = 1.5.6
```

使用过程中用到的一些命令：

```bash
#root设置密码
sudo passwd root

#安装redis
apt-get install redis
#安装memcached
apt-get install libevent-dev
apt-get install memcached

#redis启动
redis-server start
#memcached启动
memcached -d -u root

#redis性能测试工具（安装redis后自带，此次测试没有使用）
redis-benchmark -n 500 -d 10240
#安装libmemcached工具包（包括有memaslap，memslap两款memcached的性能测试工具，此次测试没有使用）
./configure --prefix=/usr/local/libmemcached --with-memcached=/usr/local/bin/memcached
make 
make install
export PATH=/usr/local/libmemcached/bin:$PATH
```

### 实验过程：

使用java客户端`Jedis`和`XMcached`进行测试，设置以下参数：

```properties
#并发线程数
thread.number=500
#一个线程需要保存的数据个数
cache.value.number=40
#每一个值的大小，单位为byte（这里设置的大小并不是准确的，例如此处值为10000，后台数据大小基本可以认为是10k）
cache.value.size=10000
```

具体的代码在[这里](https://github.com/foolish1024/cache)。

### 实验结果：

本地电脑性能不稳定，开启的应用多少都会影响实验结果，所以下面的结果没有具体耗时。

使用上述参数进行实验，发现`redis`保存数据耗时比`memcached`更短，多次实验也都是相同结果，但差距很小。

### 结论：

上述实验并没有佐证项目中出现的问题，即高并发且大数据量下`redis`性能比`memcached`差。

由于使用的两个客户端都仅仅使用的是默认参数，没有发挥出`XMemcached`客户端支持多线程的特性，所以之后又修改参数进行了测试。

结论是`XMemcached`比`Jedis`的性能略高，基本可以证明项目中出现的问题。



### 项目地址：

[redis和memcached缓存性能比较项目](https://github.com/foolish1024/cache)