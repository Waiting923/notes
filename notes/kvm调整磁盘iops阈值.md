# KVM调整磁盘iops阈值

- 这个方法只能临时生效，重启虚拟机则恢复原值，若要永久生效还需修改数据库（nova库block_device_mapping表）
1. 登陆对应计算节点，确认虚拟机
```shell
$ docker exec -it -u root nova_libvirt virsh list | grep instance-00001c43

 744   instance-00001c43              running
```
2. 查看磁盘信息
```shell
$docker exec -it -u root nova_libvirt virsh qemu-monitor-command instance-00001c43 --hmp 'info block'

drive-scsi0-0-0-0 (#block136): json:{"driver": "raw", "file": {"password-secret": "scsi0-0-0-0-secret0", "pool": "vms_hdd", "image": "volume-0c39ccb6-0eae-4535-b2ac-a1a65e2505b9", "driver": "rbd", "user": "cinder", "=keyvalue-pairs": "[\"auth_supported\", \"cephx;none\", \"mon_host\", \"10.3.13.160:6789;10.3.13.161:6789\"]"}} (raw)
    Attached to:      scsi0-0-0-0
    Cache mode:       writeback, direct
    I/O throttling:   bps=0 bps_rd=104857600 bps_wr=104857600 bps_max=0 bps_rd_max=0 bps_wr_max=0 iops=0 iops_rd=1000 iops_wr=1000 iops_max=0 iops_rd_max=0 iops_wr_max=0 iops_size=0 group=drive-scsi0-0-0-0
```
3. 根据需求数值修改
```shell
$ docker exec -it -u root nova_libvirt virsh qemu-monitor-command instance-00001c43 --hmp 'block_set_io_throttle drive-scsi0-0-0-0 0 153400320 153400320 0 1500 1500'
```
4. 查看修改结果
```shell
$ docker exec -it -u root nova_libvirt virsh qemu-monitor-command instance-00001c43 --hmp 'info block'

drive-scsi0-0-0-0 (#block136): json:{"driver": "raw", "file": {"password-secret": "scsi0-0-0-0-secret0", "pool": "vms_hdd", "image": "volume-0c39ccb6-0eae-4535-b2ac-a1a65e2505b9", "driver": "rbd", "user": "cinder", "=keyvalue-pairs": "[\"auth_supported\", \"cephx;none\", \"mon_host\", \"10.3.13.160:6789;10.3.13.161:6789\"]"}} (raw)
    Attached to:      scsi0-0-0-0
    Cache mode:       writeback, direct
    I/O throttling:   bps=0 bps_rd=153400320 bps_wr=153400320 bps_max=0 bps_rd_max=0 bps_wr_max=0 iops=0 iops_rd=1500 iops_wr=1500 iops_max=0 iops_rd_max=0 iops_wr_max=0 iops_size=0 group=drive-scsi0-0-0-0
```