# `gocache`: A distributed cache

## 简介

`gocache` 是模仿 [`groupcache`](https://github.com/golang/groupcache)（Go 语言版的 `memcached`） 实现的一个分布式缓存系统，为了将**代码量限制在 500 行左右**（`groupcache` 约 **3000 行**），裁剪了部分功能。但总体实现上，还是与 `groupcache` 非常接近的。支持特性有：

- 采用最近最少访问(Least Recently Used, LRU) 缓存策略
- 单机缓存和基于 HTTP 的分布式缓存
- 使用 Go 锁机制防止缓存击穿
- 使用一致性哈希选择节点，实现负载均衡
- 使用 `protobuf` 优化节点间二进制通信

## 编译运行

```bash
  ./run.sh
```