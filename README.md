# `gocache`: A distributed cache

## 简介

`gocache` 是模仿 [`groupcache`](https://github.com/golang/groupcache)（Go 语言版的 `memcached`） 实现的一个分布式缓存系统，为了将**代码量限制在 500 行左右**（`groupcache` 约 **3000 行**），裁剪了部分功能。支持特性有：

- 采用最近最少访问(Least Recently Used, LRU) 缓存策略
- 单机缓存和基于 HTTP 的分布式缓存
- 使用 Go 锁机制防止缓存击穿
- 使用一致性哈希选择节点，实现负载均衡
- 使用 `protobuf` 优化节点间二进制通信

主要的逻辑为：

```bash
                            是
接收 key --> 检查是否被缓存 -----> 返回缓存值
                |  否                         是
                |-----> 是否应当从远程节点获取 -----> 与远程节点交互 --> 返回缓存值
                            |  否
                            |-----> 调用`回调函数`，获取值并添加到缓存 --> 返回缓存值
```

## 目录

```go
├── README.md
├── go.mod    
├── go.sum
├── gocache
│   ├── byteview.go			  // 缓存值的抽象与封装
│   ├── cache.go			  // 并发控制
│   ├── consistenthash
│   │   ├── consistenthash.go // 一致性哈希
│   │   └── consistenthash_test.go
│   ├── go.mod
│   ├── go.sum
│   ├── gocache.go			  // 负责与外部交互，控制缓存存储和获取的主流程
│   ├── gocache_test.go
│   ├── gocachepb
│   │   ├── gocachepb.pb.go
│   │   └── gocachepb.proto
│   ├── http.go				  // 实现节点间通信
│   ├── lru
│   │   ├── lru.go			  // lru 缓存淘汰策略
│   │   └── lru_test.go
│   ├── peers.go
│   └── singleflight
│       ├── singleflight.go	  // 避免缓存击穿和穿透
│       └── singleflight_test.go
├── main.go
└── run.sh
```

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
2022/02/16 21:17:43 geecache is running at http://localhost:8001
2022/02/16 21:17:43 geecache is running at http://localhost:8002
2022/02/16 21:17:43 geecache is running at http://localhost:8003
2022/02/16 21:17:43 fontend server is running at http://localhost:9999
>>> start test
2022/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
2022/02/16 21:17:45 [Server http://localhost:8003] Pick peer http://localhost:8001
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

测试的时候，我们**并发**了 3 个请求 `?key=Tom`，从日志中可以看到，三次均选择了节点 `8001`，符合一致性哈希算法的结果。

