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
