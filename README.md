# 修改的文件

## backend.go

server/mvcc/backend/backend.go

newBackend()
```
bopts.NoSync = true
```

## wal.go

server/wal/wal.go

Create()
```
	w := &WAL{
		lg:           lg,
		dir:          dirpath,
		metadata:     metadata,
		unsafeNoSync: true,
	}
```

openAtIndex()
```
	w := &WAL{
		lg:           lg,
		dir:          dirpath,
		start:        snap,
		decoder:      newDecoder(rs...),
		readClose:    closer,
		locks:        ls,
		unsafeNoSync: true,
	}
```

## debug for Sync messages

If you want to know boltDB sync or no sync, in $GOPATH (default is ~/go)
在这个目录下，pkg/mod/go.etcd.io/bbolt@v1.3.5/boltsync_unix.go (仅测试了Mac, Linux or windows可能是其他文件)

可以修改下面的代码
```
func fdatasync(db *DB) error {
        // fmt.Println("in bolt, call fdatasync...")
        return db.file.Sync()
}
```

然后，用etcdctl做一下put的测试，证明无误后，最好马上像上面comment这个Println

注意：这是go package的工作目录，所以是临时的，而且是只读的，修改后需要overwrite，但不需要针对这个package进行go get or go install

## 未来需要增加的工作

需要修改main()，让crash不允许启动

# 测试结果

## 测试工具

用 tools/benchmark/main.go

```
go build main.go
mv main benchmark
```

受条件所限，我只能测试cluster为一台机器，所以使用 --target-leader。我们只测试写put

```
./benchmark --target-leader --conns=100 --clients=1000 put --key-size=100 --val-size=1000 --total=100000
```

ps: build server and remove db folder
```
rm -rf default.etcd |
rm -rf bin |
./build 
./bin/etcd
```

## put测试结果

注意：每次测试清理一下db folder ```rm -rf default.etcd```

### 常规 --key-size=100 --val-size=1000 --total=100000

| 版本 | conns | clients | qps | 99% latency (second) |
| -- | -- | -- | -- | -- |
| 原始 | 1 | 1 | 104.8754 | 0.0443 |
| 优化 | 1 | 1 | 5163.9124 | 0.0004 |
| 原始 | 1 | 10 | 1227.5198 | 0.0382 |
| 优化 | 1 | 10 | 20559.3612 | 0.0011 |
| 原始 | 10 | 100 | 9784.9715 | 0.0640 |
| 优化 | 10 | 100 | 26188.6270 | 0.0128 |
| 原始 | 100 | 1000 | 21718.9972 | 0.2080 |
| 优化 | 100 | 1000 | 27781.4646 | 0.2545 |

### 小key/value，--key-size=10 --val-size=100 --total=100000

| 版本 | conns | clients | qps | 99% latency (second) |
| -- | -- | -- | -- | -- |
| 原始 | 1 | 1 | 108.8595 | 0.0362 |
| 优化 | 1 | 1 | 5249.5042 | 0.0004 |
| 原始 | 1 | 10 | 1268.1578 | 0.0363 |
| 优化 | 1 | 10 | 25682.0791 | 0.0010 |
| 原始 | 10 | 100 | 11913.3493 | 0.0432 |
| 优化 | 10 | 100 | 39435.8558 | 0.0060 |
| 原始 | 100 | 1000 | 26613.6242 | 0.0984 |
| 优化 | 100 | 1000 | 39885.2402 | 0.0841 |

### 大key/value, --key-size=1000 --val-size=1000000 --total=1000

| 版本 | conns | clients | qps | 99% latency (second) |
| -- | -- | -- | -- | -- |
| 原始 | 1 | 1 | 43.4933 | 0.2034 |
| 优化 | 1 | 1 | 173.4599 | 0.1463 |
| 原始 | 1 | 10 | 62.0685 | 0.6409 |
| 优化 | 1 | 10 | 214.0351 | 0.2079 |
| 原始 | 10 | 100 | 109.3217 | 1.3296 |
| 优化 | 10 | 100 | 172.0942 | 1.6161 |

# 分析

1. 单个版本比较，可以看出，当conns和clients都很少的时候，qps比较低，对于原始，如果clients十倍的增长到100，那么qps也几乎是10倍的增长。优化版本也类似，只是增长倍数没有这么离谱。而且，不管key/value多大，如果conns=1 and clients=1，则qps都差不多，都是100这个级别。怀疑是：服务器端在做等待，收集一定量的请求(或一定时间内)，再做batch操作。

2. 原始和优化版本比较，在clients比较小时，差别最大，对于key/value=1K，以及key/value=100时，都达到了50倍。同时，99% latency只有百分之一。这是磁盘优化的最好结果。但当clients增高时，由于服务器端使用了batch的优化，这个磁盘读写优化的效果就没有那么大了，qps最低只能提高30%，99% latency也开始接近。

3. 当key/value的大小是1M，可以看出在clients=100时，其qps达到109，因为写盘是data + Raft WAL，所以其磁盘写入的带宽是200多兆，基本上是我的Mac的上限了。这时再看优化版本，由于节省了Raft WAL，所以qps还是带来了50%以上的提高。