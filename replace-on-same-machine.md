# 本机同IP替换一个新的etcd节点

## 前言

我们假设一个三节点的etcd，然后一个节点crash了，这时，我们不能直接重启这个crash机器的etcd进程，而是先从集群删除这个节点，然后以一个全新节点加入到新集群，从而保持集群可以恢复到三节点状态。

## 我们先搭建一个三节点的etcd集群

NOTE: 你需要根基你的配置，修改对应的IP地址和--data-dir=<你etcd数据库目录>

```
TOKEN=token-01
CLUSTER_STATE=new
NAME_1=machine-1
NAME_2=machine-2
NAME_3=machine-3
HOST_1=192.168.0.11
HOST_2=192.168.0.22
HOST_3=192.168.0.33
CLUSTER=${NAME_1}=http://${HOST_1}:2380,${NAME_2}=http://${HOST_2}:2380,${NAME_3}=http://${HOST_3}:2380
```

For machine 1:
```
THIS_NAME=${NAME_1}
THIS_IP=${HOST_1}
./etcd --data-dir=/root/br_rocksdb/data.etcd --name ${THIS_NAME} \
--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
--initial-cluster ${CLUSTER} \
--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
```

For machine 2:
```
THIS_NAME=${NAME_2}
THIS_IP=${HOST_2}
./etcd --data-dir=/root/br_rocksdb/data.etcd --name ${THIS_NAME} \
--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
--initial-cluster ${CLUSTER} \
--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
```

For machine 3:
```
THIS_NAME=${NAME_3}
THIS_IP=${HOST_3}
./etcd --data-dir=/root/br_rocksdb/data.etcd --name ${THIS_NAME} \
--initial-advertise-peer-urls http://${THIS_IP}:2380 --listen-peer-urls http://${THIS_IP}:2380 \
--advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379 \
--initial-cluster ${CLUSTER} \
--initial-cluster-state ${CLUSTER_STATE} --initial-cluster-token ${TOKEN}
```

## 检查集群状态工具

Use etcdctl to check 
```
export ETCDCTL_API=3
HOST_1=192.168.0.11
HOST_2=192.168.0.22
HOST_3=192.168.0.33
ENDPOINTS=$HOST_1:2379,$HOST_2:2379,$HOST_3:2379
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

## 替换步骤

注意：必须用新的机器名，IP可以一样。如果使用旧的机器名，会出现一些异常，不推荐

### 第一个步：删除meta data（假设crash node IP是192.168.0.11）

Remove one node from cluster (e.g. 192.168.0.11)，这个是删除meta data，可以任何机器上执行

1. get member id
```
./etcdctl --endpoints=${HOST_1}:2379,${HOST_2}:2379,${HOST_3}:2379 member list
```

2. remove the member (NOTE: replace the correct MEMBER_ID which is from the above list)
```
MEMBER_ID=c37e506cd75ed722
./etcdctl --endpoints=${HOST_1}:2379,${HOST_2}:2379,${HOST_3}:2379 \
member remove ${MEMBER_ID}
```

### 第二步：加入一个新机器名到集群

这个是更新etcd cluster的meta data，实际新机器还没有启动（你可以用最下面的检查工具查看集群状态）

3. add a new node 4 (in the same IP of 192.168.0.11)

```
NAME_4=machine-4
HOST_4=192.168.0.11
./etcdctl --endpoints=${HOST_2}:2379,${HOST_3}:2379 \
member add ${NAME_4} \
--peer-urls=http://${HOST_4}:2380
```

### 第三步：清除磁盘目录

4. rm the old data folder for the ssame machine (192.168.0.11)
```
rm -rf /root/br_rocksdb/data.etcd
```

### 第四步：启动新进程

5. start a new member (in the same IP) to existing cluster
```
TOKEN=token-01
CLUSTER_STATE=existing
NAME_2=machine-2
NAME_3=machine-3
NAME_4=machine-4
HOST_2=192.168.0.22
HOST_3=192.168.0.33
HOST_4=192.168.0.11
CLUSTER=${NAME_2}=http://${HOST_2}:2380,${NAME_3}=http://${HOST_3}:2380,${NAME_4}=http://${HOST_4}:2380
THIS_NAME=${NAME_4}
THIS_IP=${HOST_4}
./etcd --data-dir=/root/br_rocksdb/data.etcd --name ${THIS_NAME} \
--initial-advertise-peer-urls http://${THIS_IP}:2380 \
--listen-peer-urls http://${THIS_IP}:2380 \
--advertise-client-urls http://${THIS_IP}:2379 \
--listen-client-urls http://${THIS_IP}:2379 \
--initial-cluster ${CLUSTER} \
--initial-cluster-state ${CLUSTER_STATE} \
--initial-cluster-token ${TOKEN}
```

### 再次检查

6. check
```
./etcdctl --write-out=table --endpoints=$ENDPOINTS endpoint status
```

### 其他帮助

如果重新启动一个crash的etcd node（注意：不能是优化版本），用下面的命令

```
./etcd --data-dir=/root/br_rocksdb/data.etcd --initial-cluster-token ${TOKEN} --name ${THIS_NAME} --listen-peer-urls http://${THIS_IP}:2380 --advertise-client-urls http://${THIS_IP}:2379 --listen-client-urls http://${THIS_IP}:2379
```