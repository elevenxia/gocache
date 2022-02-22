# `gocache`: A distributed cache

## 简介

`gocache` 是模仿 [`groupcache`](https://github.com/golang/groupcache)（Go 语言版的 `memcached`） 实现的一个分布式缓存系统，为了将**代码量限制在 500 行左右**（`groupcache` 约 **3000 行**），裁剪了部分功能。但总体实现上，还是与 `groupcache` 非常接近的。支持特性有：

- 采用最近最少访问(Least Recently Used, LRU) 缓存策略
- 单机缓存和基于 HTTP 的分布式缓存
- 使用 Go 锁机制防止缓存击穿
- 使用一致性哈希选择节点，实现负载均衡
- 使用 `protobuf` 优化节点间二进制通信

## 编译运行


为了方便，我们将启动的命令封装为一个 `shell` 脚本（`run.sh`）：

```bash
#!/bin/bash
trap "rm server;kill 0" EXIT

go build -o server
./server -port=8001 &
./server -port=8002 &
./server -port=8003 -api=1 &

sleep 2
echo ">>> start test"
curl "http://localhost:9999/api?key=Tom" &
curl "http://localhost:9999/api?key=Tom" &
curl "http://localhost:9999/api?key=Tom" &

wait
```

- `trap` 命令用于在 shell 脚本退出时，删掉临时文件，结束子进程。

```bash
$ ./run.sh
2020/02/16 21:17:43 geecache is running at http://localhost:8001
2020/02/16 21:17:43 geecache is running at http://localhost:8002
2020/02/16 21:17:43 geecache is running at http://localhost:8003
2020/02/16 21:17:43 fontend server is running at http://localhost:9999
>>> start test
2020/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
2020/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
2020/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
...
630630630
```

此时，我们可以打开一个新的 shell，进行测试：

```bash
$ curl "http://localhost:9999/api?key=Tom"
630
$ curl "http://localhost:9999/api?key=kkk"
kkk not exist
```

测试的时候，我们**并发**了 3 个请求 `?key=Tom`，从日志中可以看到，三次均选择了节点 `8001`，这是一致性哈希算法的功劳。
