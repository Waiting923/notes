# ceph osd下线操作

确认osd是否已经踢出集群（down/out）

若未踢出集群则手动操作停止osd进程并踢出集群，操作前可调整ceph集群均衡速率以减轻数据均衡时对集群读写的影响
```
sudo ceph tell osd.* injectargs --osd_recovery_sleep 0.1 #数值范围0~0.5 数值越大均衡速率越慢，影响越小
```

停止osd进程
```
systemctl stop ceph-osd@571
```

清理挂载点
```
sudo umount /var/lib/ceph/osd/ceph-571
```

等待超时自动out，或手动out
```
ceph osd out 571
```

若立即更换新盘则无需删除crushmap中osd信息

12版本以后还需清理lvm

使用lvremove/vgremove/pvremove清理相关信息

检查raid卡cache
```
sudo /opt/MegaRAID/MegaCli/MegaCli64  -GetPreservedCacheList -aALL
Adapter 0: No Virtual Drive has Preserved Cache Data.
Exit Code: 0x00
```

若存在cache则根据cache信息清理raid cache，不清理有可能导致新创建的raid存在id被占用
```
sudo /opt/MegaRAID/MegaCli/MegaCli64 -DiscardPreservedCache -L1 -a0
```

查询残留dm信息，防止新插磁盘盘符被占用与原盘符不符
```
mount | grep 571
/dev/mapper/ceph--25f16781--66cb--4fbc--ad9e--f79431d189af-osd--data--5b8dcf95--ff39--4457--aeb7--4748878d5f88 on /var/lib/ceph/osd/ceph-571 type xfs (rw,noatime,attr2,inode64,noquota)
/dev/mapper/ceph--571934bf--2332--4bda--93ac--cc4d4ef5f185-osd--data--5d1b2bb3--c2d7--4164--aa0d--2e406901114a on /var/lib/ceph/osd/ceph-581 type xfs (rw,noatime,attr2,inode64,noquota)

ll /dev/disk/by-id/ | grep ceph--25f16781--66cb--4fbc--ad9e--f79431d189af-osd--data--5b8dcf95--ff39--4457--aeb7--4748878d5f88
lrwxrwxrwx 1 root root 11 Sep  8  2018 dm-name-ceph--25f16781--66cb--4fbc--ad9e--f79431d189af-osd--data--5b8dcf95--ff39--4457--aeb7--4748878d5f88 -> ../../dm-12

sudo dmsetup ls | grep ceph--25f16781--66cb--4fbc--ad9e--f79431d189af-osd--data--5b8dcf95--ff39--4457--aeb7--4748878d5f88
ceph--25f16781--66cb--4fbc--ad9e--f79431d189af-osd--data--5b8dcf95--ff39--4457--aeb7--4748878d5f88      (253:12)

sudo ls -l /sys/devices/virtual/block/dm-12/slaves/
total 0
lrwxrwxrwx 1 root root 0 Dec 31 09:56 sdk2 -> ../../../../pci0000:3a/0000:3a:02.0/0000:3c:00.0/host0/target0:0:10/0:0:10:0/block/sdk/sdk2
```

清理dm信息
```
sudo dmsetup remove ceph--25f16781--66cb--4fbc--ad9e--f79431d189af-osd--data--5b8dcf95--ff39--4457--aeb7--4748878d5f88
```

硬盘数据擦除,需要故障确认时记录的Enclosure Device ID与Device Id以及raid卡id
```
sudo /opt/MegaRAID/MegaCli/MegaCli64 -SecureErase Start Normal -PhysDrv [32:10] -a0
```

擦除进度查询
```
sudo /opt/MegaRAID/MegaCli/MegaCli64 -SecureErase ShowProg -PhysDrv [32:10] -a0
```
