# ceph osd 故障硬盘更换后重新上线操作

确认更换后的磁盘信息（查看是否是新盘）

命令参考[这里](https://github.com/Waiting923/notes/blob/master/notes/ceph%20osd硬盘故障定位.md)的smartctl命令

确认磁盘正常识别，盘符识别正确，调整磁盘raid模式

配置raid0操作
```
#设置raid-ready
$ sudo /opt/MegaRAID/MegaCli/MegaCli64 -PDMakeGood -PhysDrv[32:10] -force -a0
Adapter: 0: EnclId-8 SlotId-5 state changed to Unconfigured-Good.
Exit Code: 0x00

#创建raid0
$ sudo /opt/MegaRAID/MegaCli/MegaCli64 -CfgLdAdd -r0[32:10] WB NORA Cached -strpsz64 -a0
Adapter 0: Created VD 2
Adapter 0: Configured the Adapter!!
Exit Code: 0x00

#开启cache（这里的VD_ID：2由上一步结果输出）
$ sudo /opt/MegaRAID/MegaCli/MegaCli64 -LDSetProp -EnDskCache -L2 -a0
Set Disk Cache Policy to Enabled on Adapter 0, VD 2 (target id: 2) success
Exit Code: 0x00
```

配置JBOD操作
```
$ sudo /opt/MegaRAID/MegaCli/MegaCli64 -PDMakeJBOD -PhysDrv[32:0] -force -a0
```

若ceph journal盘与数据盘复用，则进行分区
```
$ sudo parted /dev/sdk mklabel gpt
$ sudo parted /dev/sdk mkpart ceph_journal 0% 30G
$ sudo parted /dev/sdk mkpart ceph_data 30G 100%

如果分区已存在，可使用wipefs清除残留FileSystem信息
$ sudo wipefs -a /dev/sdk1
```

在创建osd前先确认集群均衡速率已经调整
```
$ ceph daemon mon.{mon_name} config show | grep osd_recovery_sleep
```

磁盘确认后操作ceph，删除该osd认证权限以及osd,之所以放在这里删除，是为了防止同时删除多各osd但osd加入集群的顺序又不同，导致osd id乱序的问题。
```
$ ceph osd auth del 571
$ ceph osd rm 571
```

初始化osd，根据实际journal分区分配情况
```
$ sudo ceph-volume lvm create --filestore --data /dev/sdk1 --journal /dev/sdk2
```

初始化后osd应该为up/out状态，手动in osd
```
$ ceph osd in 571
```

等待均衡完成
