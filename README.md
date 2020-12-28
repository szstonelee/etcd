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
