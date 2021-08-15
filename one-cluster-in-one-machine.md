# 在一台机器上搭建三节点etcd集群并用于测试（未完成）

此测试当前还不可行，需要在更多核的机器上测试上再考虑，因为当前测试，用ramfs，比之前用SSD数据还差，怀疑是一个机器上跑三个etcd加一个benchmark，瓶颈在CPU上。

## 操作命令

常见ramfs
```
cd
mkdir ramfs
mount -t ramfs -o size=3G ramfs ~/ramfs
```


```
TOKEN=token-one-cluster-in-one-machine
CLUSTER_STATE=new
NAME_1=machine-1
NAME_2=machine-2
NAME_3=machine-3
HOST_1=192.168.0.11
HOST_2=192.168.0.11
HOST_3=192.168.0.11
CLUSTER=${NAME_1}=http://${HOST_1}:12380,${NAME_2}=http://${HOST_2}:22380,${NAME_3}=http://${HOST_3}:32380
```

进程一
```
./etcd --data-dir=/root/ramfs/data1.etcd --name ${NAME_1} \
--initial-advertise-peer-urls http://${HOST_1}:12380 --listen-peer-urls http://${HOST_1}:12380 \
--advertise-client-urls http://${HOST_1}:12379 --listen-client-urls http://${HOST_1}:12379 \
--initial-cluster ${CLUSTER} \
--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN} --unsafe-no-fsync
```

进程二
```
./etcd --data-dir=/root/ramfs/data2.etcd --name ${NAME_2} \
--initial-advertise-peer-urls http://${HOST_2}:22380 --listen-peer-urls http://${HOST_2}:22380 \
--advertise-client-urls http://${HOST_2}:22379 --listen-client-urls http://${HOST_2}:22379 \
--initial-cluster ${CLUSTER} \
--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN} --unsafe-no-fsync
```

进程三
```
./etcd --data-dir=/root/ramfs/data3.etcd --name ${NAME_3} \
--initial-advertise-peer-urls http://${HOST_3}:32380 --listen-peer-urls http://${HOST_3}:32380 \
--advertise-client-urls http://${HOST_3}:32379 --listen-client-urls http://${HOST_3}:32379 \
--initial-cluster ${CLUSTER} \
--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN} --unsafe-no-fsync
```

检查状态
```
export ETCDCTL_API=3
HOST_1=192.168.0.11
HOST_2=192.168.0.11
HOST_3=192.168.0.11
ENDPOINTS=$HOST_1:12379,$HOST_2:22379,$HOST_3:32379
```

member list
```
./etcdctl --endpoints=$ENDPOINTS member list
```

测试写
```
./etcdctl --endpoints=$ENDPOINTS put foo "Hello World!"
./etcdctl --endpoints=$ENDPOINTS get foo
./etcdctl --endpoints=$ENDPOINTS --write-out="json" get foo
```

cluster status and healthy (and member list)
```
./etcdctl --write-out=table --endpoints=$ENDPOINTS endpoint status
./etcdctl --endpoints=$ENDPOINTS endpoint health
./etcdctl --endpoints=$ENDPOINTS member list
```

## Benchamrk

### 命令原型
```
./benchmark --endpoints=$ENDPOINTS --conns=1 --clients=1 put --key-size=100 --val-size=1000 --total=100000 key-space-size=1000000
```

下面的测试中，请替换相关参数，包括：conns, clients, key-size, val-size

压缩磁盘辅助命令（如果某个测试或某些测试导致库大小可能超过2G的话）
```
rev=$(ETCDCTL_API=3 ./etcdctl --endpoints=$ENDPOINTS endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9]*')
echo $rev
# 通过上面的rev里的数字(三个数值应该一样)，输入到下面的值<rev值>
./etcdctl --endpoints=$ENDPOINTS compact <rev值>
./etcdctl --endpoints=$ENDPOINTS defrag --cluster
./etcdctl --endpoints=$ENDPOINTS alarm disarm
```

### 常规 (--key-size=100 --val-size=1000)

| conn | clients | 单机集群qps |  网络集群参考qps | 提升度 |
| -- | -- | -- | -- | -- |
| 1 | 1 | 2016 | 0 | 0% |
| 1 | 4 | 3886 | 0 | 0% |
| 1 | 16 | 7091 | 0 | 0% |
| 10 | 10 | 0 | 0 | 0% |
| 10 | 40 | 0 | 0 | 0% |
| 10 | 160 | 0 | 0 | 0% |
| 100 | 100 | 0 | 0 | 0% |
| 100 | 400 | 0 | 0 | 0% |
| 100 | 1600 | 0 | 0 | 0% |

### 小值 (--key-size=10 --val-size=100)

| conn | clients | 单机集群qps |  网络集群参考qps | 提升度 |
| -- | -- | -- | -- | -- |
| 1 | 1 | 0 | 0 | 0% |
| 1 | 4 | 0 | 0 | 0% |
| 1 | 16 | 0 | 0 | 0% |
| 10 | 10 | 0 | 0 | 0% |
| 10 | 40 | 0 | 0 | 0% |
| 10 | 160 | 0 | 0 | 0% |
| 100 | 100 | 0 | 0 | 0% |
| 100 | 400 | 0 | 0 | 0% |
| 100 | 1600 | 0 | 0 | 0% |

### 大值 (--key-size=100 --val-size=10000)

| conn | clients | 单机集群qps |  网络集群参考qps | 提升度 |
| -- | -- | -- | -- | -- |
| 1 | 1 | 0 | 0 | 0% |
| 1 | 4 | 0 | 0 | 0% |
| 1 | 16 | 0 | 0 | 0% |
| 10 | 10 | 0 | 0 | 0% |
| 10 | 40 | 0 | 0 | 0% |
| 10 | 160 | 0 | 0 | 0% |
| 100 | 100 | 0 | 0 | 0% |
| 100 | 400 | 0 | 0 | 0% |
| 100 | 1600 | 0 | 0 | 0% |
