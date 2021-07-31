# 三机集群测试

## 测试环境

搭建一个三集群的etcd的相关命令，[可参考这里](replace-on-same-machine.md)

* 每个机器2vCPU(Intel Cascade Lake 3.0GHz), 内存7.6G，本地双SSD(各50G)，其中一SSD为etcd存储独享，网卡千兆
* etcd 3.5(preview上改造和优化) and etcd 3.5正式版
* CentOs7 2台，CentOS8 1台

仅测试put，key-space-size = 1000000，totoal = 100K，不用--target-leader

key和value的大小分三种情况
* 常规：key size = 100 bytes, value size = 1K bytes
* 小值：key size = 10 bytes, value size = 100 bytes
* 大值：key size = 100 bytes, value size = 10K bytes

然后分别测试: connection在1，10， 100时，clients对应为1，4，16倍（即1 HTTP2 connection复用1个、4个、16个gRPC client）

测试命令原型
```
./benchmark --endpoints=$ENDPOINTS --conns=1 --clients=1 put --key-size=100 --val-size=1000 --total=100000 key-space-size=1000000
```

## 一些辅助命令工具

由于etcd缺省是2G磁盘配额(quota)，超过了，系统会给出warning，所以注入了一定量的key/value后，需要压缩磁盘（benchmark每次都自动删除所有的key，但磁盘未释放），方法如下：

压缩磁盘
```
rev=$(ETCDCTL_API=3 ./etcdctl --endpoints=$ENDPOINTS endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')
echo $rev
# 通过上面的rev里的数字(三个数值应该一样)，输入到下面的值<rev值>
./etcdctl --endpoints=$ENDPOINTS compact <rev值>
./etcdctl --endpoints=$ENDPOINTS defrag --cluster
./etcdctl --endpoints=$ENDPOINTS alarm disarm
```

最后check 磁盘（应该只有百兆，之前是2G以上）
```
du -h -d 0 /root/br_rocksdb/data.etcd
./etcdctl --write-out=table --endpoints=$ENDPOINTS endpoint status
```
通过etcdctl，应该看到集群里所有的node的db size都很小，而且ERRORS是空

下面的删除和检查是否存在key权当帮助，实际用不上：

删除所有的key
```
./etcdctl --endpoints=$ENDPOINTS del "*" --from-key=true
```

检查Databas里是否还有key
```
./etcdctl --endpoints=$ENDPOINTS get "*" --from-key=true
```


## 常规 (--key-size=100 --val-size=1000)

| 版本 | conn | clients | qps | P90(ms) | P99(ms) | Throughput提升度 |
| -- | -- | -- | -- | -- | -- | -- |
| 原始 | 1 | 1 | 1444 | 0.6 | 7.8 | |
| 优化 | 1 | 1 | 1482 | 0.6 | 6.1 | 3% |
| 原始 | 1 | 4 | 2792 | 2.2 | 11.5 | |
| 优化 | 1 | 4 | 3858 | 1.1 | 9.6 | 38% |
| 原始 | 1 | 16 | 5774 | 6.2 | 14.2 | |
| 优化 | 1 | 16 | 8854 | 2.8 | 11.4 | 53% |
| 原始 | 10 | 10 | 6229 | 2.3 | 6.5 | |
| 优化 | 10 | 10 | 9478 | 1.4 | 4.9 | 52% |
| 原始 | 10 | 40 | 12758 | 4.7 | 11.0 | |
| 优化 | 10 | 40 | 17534 | 3.8 | 10.3 | 37% |
| 原始 | 10 | 160 | 18038 | 13.6 | 21.8 | |
| 优化 | 10 | 160 | 24446 | 11.4 | 24.2 | 36% |
| 原始 | 100 | 100 | 14768 | 9.7 | 17.3 | |
| 优化 | 100 | 100 | 19503 | 8.5 | 18.6 | 32% |
| 原始 | 100 | 400 | 19801 | 28.6 | 43.1 | |
| 优化 | 100 | 400 | 24956 | 25.8 | 89.4 | 26% |
| 原始 | 100 | 1600 | 23215 | 95.0 | 135.6 | |
| 优化 | 100 | 1600 | 28700 | 91.7 | 257.9 | 24% |

## 小值 (--key-size=10 --val-size=100)

| 版本 | conn | clients | qps | P90(ms) | P99(ms) | Throughput提升度 |
| -- | -- | -- | -- | -- | -- | -- |
| 原始 | 1 | 1 | 1674 | 0.6 | 6.0 |  |
| 优化 | 1 | 1 | 1570 | 0.6 | 5.5 | -6% |
| 原始 | 1 | 4 | 3471 | 1.5 | 10.6 | |
| 优化 | 1 | 4 | 4190 | 1.0 | 9.2 | 21% |
| 原始 | 1 | 16 | 7102 | 4.5 | 13.0 | |
| 优化 | 1 | 16 | 9627 | 2.4 | 11.2 | 36% |
| 原始 | 10 | 10 | 7822 | 1.7 | 3.4 | |
| 优化 | 10 | 10 | 10355 | 1.3 | 4.8 | 32% |
| 原始 | 10 | 40 | 14848 | 3.9 | 9.5 | |
| 优化 | 10 | 40 | 20345 | 3.0 | 10.0 | 37% |
| 原始 | 10 | 160 | 22562 | 10.6 | 17.5 | |
| 优化 | 10 | 160 | 30533 | 9.0 | 18.7 | 35% |
| 原始 | 100 | 100 | 17250 | 8.2 | 14.1 | |
| 优化 | 100 | 100 | 22982 | 7.0 | 16.1 | 33% |
| 原始 | 100 | 400 | 23466 | 24.3 | 39.7 | |
| 优化 | 100 | 400 | 31789 | 20.4 | 46.2 | 35% |
| 原始 | 100 | 1600 | 29514 | 75.5 | 101.7 | |
| 优化 | 100 | 1600 | 37516 | 74.1 | 123.7 | 27% |

## 大值 (--key-size=100 --val-size=10000)

注意：
1. 由于etcd给的磁盘量是2G，所以每次测试完需要defrag，
2. 我们不测试4， 16倍clients，因为超时（可能是包太大）

| 版本 | conn | clients | qps | P90(ms) | P99(ms) | Throughput提升度 |
| -- | -- | -- | -- | -- | -- | -- |
| 原始 | 1 | 1 | 946 | 1.2 | 10.2 | |
| 优化 | 1 | 1 | 1340 | 0.7 | 6.7 | 42% |
| 原始 | 10 | 10 | 2719 | 4.9 | 55.2 | |
| 优化 | 10 | 10 | 3722 | 2.1 | 10.7 | 37% |
| 原始 | 100 | 100 | 3568 | 100.8 | 127.3 | |
| 优化 | 100 | 100 | 4059 | 14.3 | 257.1 | 14% |
