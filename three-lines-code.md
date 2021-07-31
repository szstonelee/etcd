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

## verify for Sync messages

我们为了证实修改有效，可以加入一些debug ouptut log信息来证实。

If you want to know boltDB sync or no sync, in $GOPATH (default is ~/go)

在这个目录下，pkg/mod/go.etcd.io/bbolt@v1.3.5/boltsync_unix.go (仅测试了Mac, Linux or windows可能是其他文件)

可以修改下面的代码
```
func fdatasync(db *DB) error {
        // fmt.Println("in bolt, call fdatasync...")    // verify the above code is effective
        return db.file.Sync()
}
```

然后，用etcdctl做一下put的测试，证明无误后（单机集群下，不会出现上面的Println信息），最好马上像上面comment这个Println

注意：这是go package的工作目录，所以是临时的，而且是只读的，修改后需要overwrite，但不需要针对这个package进行go get or go install
