# fio校验数据一致性

在测试cinder multi-attach功能时发现，虽然cinder通过multi-attach功能提供了hardware层面的disk shared,
但是在instance system层面仍然需要通过共享文件系统来进行共享磁盘的挂载和使用，主要是因为需要依赖于共享文件
系统的文件锁，这样才能保证在两个文件系统同时对于块设备中同一区域进行读写时候的数据一致性。

调研后发现GFS2（redhat global file system,不是google file system）和OCFS可以提供该功能，于是决定在instance
上使用GFS2进行测试，GFS2需要依赖于HA集群的lock,便搭建pacemaker+clvm+gfs2的架构来使用，搭建后发现pacemaker
的fence agent并没有可以适用于instance的agent,就更换另外一种思路，不使用文件系统直接对块设备进行读写验证数据一致
性，就使用到了fio。

- 使用fio写入特定数据并进行检验（如下命令我在sdb的2~4G区域使用aaaa写入,并使用crc32c进行数据检验）

```
fio -filename=/dev/sdb -direct=1 -iodepth 32 -thread -rw=write -ioengine=psync -bs=4k -offset=2G -size=2G -verify=crc32c -verify_pattern=0xAA -numjobs=1 -runtime=60 -group_reporting -name=test-pt
```

- 使用hexdump分别从两台机器对该磁盘的2~4G区域数据的第一个4K block进行数据检查
```
hexdump -s 2G -n 4k -v /dev/sdb
```

检查结果如下图所示，可以看出两台机器所显示该4k block的head校验值一致，填写数据aaaa一致，由此可以证明数据在两台机器中写入是同步一致的。

![hexdump1.png](https://github.com/Riverdd/picture/blob/master/hexdump1.png)

![hexdump2.png](https://github.com/Riverdd/picture/blob/master/hexdump2.png)

