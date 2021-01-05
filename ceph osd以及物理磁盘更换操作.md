#ceph osd以及物理磁盘更换操作
##ceph准备操作
- 确认故障osd对应盘符（以osd.571 raid卡掉盘故障为例）
```
查找osd所在节点
ceph osd find 571
{
    "osd": 571,
    "ip": "***.***.***.***:6820/18008",
    "crush_location": {
        "host": "**************",
        "rack": "*********************",
        "root": "*********************"
    }
}

登陆该节点确认osd盘符
ceph10版本命令
ceph-disk list

ceph12版本命令
ceph-volume lvm list

===== osd.572 ======

  [data]    /dev/ceph-fff2e308-6deb-49c3-91ad-2d92e871e417/osd-data-bb38f950-a15e-4f1c-aade-14e529f45db6

      type                      data
      journal uuid              8161c521-10dc-4f34-9a4d-0ed92341133c
      osd id                    572
      cluster fsid              de6a410c-e1b1-4070-a655-a401fb5272ec
      cluster name              ceph
      osd fsid                  bb38f950-a15e-4f1c-aade-14e529f45db6
      encrypted                 0
      data uuid                 WHvGBd-663F-OVfh-ZZ9V-SAZ3-UdjR-SpSHdw
      cephx lockbox secret
      crush device class        None
      data device               /dev/ceph-fff2e308-6deb-49c3-91ad-2d92e871e417/osd-data-bb38f950-a15e-4f1c-aade-14e529f45db6
      vdo                       0
      journal device            /dev/sdl1
      devices                   /dev/sdl2

  [journal]    /dev/sdl1

      PARTUUID                  8161c521-10dc-4f34-9a4d-0ed92341133c

===== osd.570 ======

  [data]    /dev/ceph-b41c626e-d897-40fd-b16b-095c63de0bfa/osd-data-eec68727-c3f3-48f5-a4b5-ca7b2c8193cb

      type                      data
      journal uuid              7aaf37fb-844d-4d7a-b968-c2eda2b9b6df
      osd id                    570
      cluster fsid              de6a410c-e1b1-4070-a655-a401fb5272ec
      cluster name              ceph
      osd fsid                  eec68727-c3f3-48f5-a4b5-ca7b2c8193cb
      encrypted                 0
      data uuid                 GKzEXe-mqrm-sd16-h1iQ-yiPY-Rzn7-Lbra8E
      cephx lockbox secret
      crush device class        None
      data device               /dev/ceph-b41c626e-d897-40fd-b16b-095c63de0bfa/osd-data-eec68727-c3f3-48f5-a4b5-ca7b2c8193cb
      vdo                       0
      journal device            /dev/sdj1
      devices                   /dev/sdj2

  [journal]    /dev/sdj1

      PARTUUID                  7aaf37fb-844d-4d7a-b968-c2eda2b9b6df

osd571已经从raid卡掉盘，所以使用排除法确认盘符以及日志盘符为sdk1,sdk2

确认raid卡下的target/device id
如下所示link名称pci部分代表pci总线地址，pci总线地址按照raid卡序号顺序排列，所以ll所显示从上到下分别对应raid卡0，1，2...
scsi部分代表改磁盘对应在raid卡上的targetid(若做raid)或deviceid(若jbod)
这里通过推断osd.571对应的物理盘符sdk对应的是raid卡0上的device id 10（这里磁盘为ssd没有做raid）
ll /dev/disk/by-path/
lrwxrwxrwx 1 root root  9 Sep  8  2018 pci-0000:3c:00.0-scsi-0:0:9:0 -> ../../sdj
lrwxrwxrwx 1 root root 10 Sep  8  2018 pci-0000:3c:00.0-scsi-0:0:9:0-part1 -> ../../sdj1
lrwxrwxrwx 1 root root 10 Sep  8  2018 pci-0000:3c:00.0-scsi-0:0:9:0-part2 -> ../../sdj2
lrwxrwxrwx 1 root root  9 Sep  8  2018 pci-0000:3c:00.0-scsi-0:0:11:0 -> ../../sdl
lrwxrwxrwx 1 root root 10 Sep  8  2018 pci-0000:3c:00.0-scsi-0:0:11:0-part1 -> ../../sdl1
lrwxrwxrwx 1 root root 10 Sep  8  2018 pci-0000:3c:00.0-scsi-0:0:11:0-part2 -> ../../sdl2

查询raid卡上该盘对应信息
jbod模式执行下面命令查询
sudo /opt/MegaRAID/MegaCli/MegaCli64 -PdList -a0 -NoLog
因为我们的osd.571(raid-0-10)已经掉盘，则以后一块盘为例
Enclosure Device ID: 32      #需要记录
Slot Number: 11    #物理插槽编号                          
Enclosure position: 1
Device Id: 11               #需要记录
WWN: 55cd2e414f860b2c
Sequence Number: 2
Media Error Count: 0          #坏道数量(重点关注)
Other Error Count: 24331      #磁盘可能存在松动（非重点关注）
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 3.492 TB [0x1bf1f72b0 Sectors]
Non Coerced Size: 3.492 TB [0x1bf0f72b0 Sectors]
Coerced Size: 3.492 TB [0x1bf0c0000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: JBOD         #对应raid模式
Device Firmware Level: 0142
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b33395e4cb
Connected Port Number: 0(path0)
Inquiry Data: PHYS8212016D3P8EGN  INTEL SSDSC2KB038T7                     SCV10142  #SN以及型号
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None
Device Speed: 6.0Gb/s
Link Speed: 6.0Gb/s
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature :21C (69.80 F)
PI Eligibility:  No
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s
Drive has flagged a S.M.A.R.T alert : No

若做了raid的磁盘上面命令同样可以查询PD信息，执行下面命令查询更为清晰看出VD信息
sudo /opt/MegaRAID/MegaCli/MegaCli64 -LdPdInfo -a0 -NoLog

Adapter #0

Number of Virtual Disks: 1
Virtual Drive: 24 (Target Id: 24)
Name                :
RAID Level          : Primary-1, Secondary-0, RAID Level Qualifier-0
Size                : 744.625 GB
Sector Size         : 512
Is VD emulated      : Yes
Mirror Data         : 744.625 GB
State               : Optimal
Strip Size          : 64 KB
Number Of Drives    : 2
Span Depth          : 1
Default Cache Policy: WriteThrough, ReadAheadNone, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteThrough, ReadAheadNone, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Enabled
Encryption Type     : None
Default Power Savings Policy: Controller Defined
Current Power Savings Policy: None
Can spin up in 1 minute: No
LD has drives that support T10 power conditions: No
LD's IO profile supports MAX power savings with cached writes: No
Bad Blocks Exist: No
Is VD Cached: No
Number of Spans: 1
Span: 0 - Number of PDs: 2

PD: 0 Information
Enclosure Device ID: 32
Slot Number: 24
Drive's position: DiskGroup: 0, Span: 0, Arm: 0
Enclosure position: 1
Device Id: 24
WWN: 55cd2e414e0d31af
Sequence Number: 2
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 745.211 GB [0x5d26ceb0 Sectors]
Non Coerced Size: 744.711 GB [0x5d16ceb0 Sectors]
Coerced Size: 744.625 GB [0x5d140000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: Online, Spun Up
Device Firmware Level: DL43
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b33395e4d8
Connected Port Number: 0(path0)
Inquiry Data:   PHDV724302DY800CGNSSDSC2BB800G7R                          N201DL43
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None
Device Speed: 6.0Gb/s
Link Speed: 6.0Gb/s
Media Type: Solid State Device
Drive Temperature :27C (80.60 F)
PI Eligibility:  No
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s
Drive has flagged a S.M.A.R.T alert : No




PD: 1 Information
Enclosure Device ID: 32
Slot Number: 25
Drive's position: DiskGroup: 0, Span: 0, Arm: 1
Enclosure position: 1
Device Id: 25
WWN: 55cd2e414e0d31ce
Sequence Number: 2
Media Error Count: 0
Other Error Count: 0
Predictive Failure Count: 0
Last Predictive Failure Event Seq Number: 0
PD Type: SATA

Raw Size: 745.211 GB [0x5d26ceb0 Sectors]
Non Coerced Size: 744.711 GB [0x5d16ceb0 Sectors]
Coerced Size: 744.625 GB [0x5d140000 Sectors]
Sector Size:  512
Logical Sector Size:  512
Physical Sector Size:  4096
Firmware state: Online, Spun Up
Device Firmware Level: DL43
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500056b33395e4d9
Connected Port Number: 0(path0)
Inquiry Data:   PHDV724302EV800CGNSSDSC2BB800G7R                          N201DL43
FDE Capable: Not Capable
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None
Device Speed: 6.0Gb/s
Link Speed: 6.0Gb/s
Media Type: Solid State Device
Drive Temperature :28C (82.40 F)
PI Eligibility:  No
Drive is formatted for PI information:  No
PI: No PI
Drive's NCQ setting : N/A
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s
Drive has flagged a S.M.A.R.T alert : No




Exit Code: 0x00

通过smartctl再次确认磁盘故障
查询raid卡disk信息
sudo smartctl --scan
/dev/sdj -d scsi # /dev/sdj, SCSI device（JBOD）
/dev/bus/0 -d megaraid,11 # /dev/bus/0 [megaraid_disk_11], SCSI device （RAID）

确认raid卡对应的bus以及device id后查询磁盘信息
sudo smartctl -a /dev/sdj（JBOD）
smartctl 6.2 2017-02-27 r4394 [x86_64-linux-3.10.0-693.21.1.el7.x86_64] (local build)
Copyright (C) 2002-13, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Device Model:     INTEL SSDSC2KB038T7
Serial Number:    PHYS821200K93P8EGN
LU WWN Device Id: 5 5cd2e4 14f85f85e
Firmware Version: SCV10142
User Capacity:    3,840,755,982,336 bytes [3.84 TB]
Sector Sizes:     512 bytes logical, 4096 bytes physical
Rotation Rate:    Solid State Device
Device is:        Not in smartctl database [for details use: -P showall]
ATA Version is:   ACS-3 (unknown minor revision code: 0x006d)
SATA Version is:  SATA >3.1, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Tue Jan  5 11:27:00 2021 CST
SMART support is: Available - device has SMART capability.
SMART support is: Enabled

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

General SMART Values:
Offline data collection status:  (0x00) Offline data collection activity
                                        was never started.
                                        Auto Offline Data Collection: Disabled.
Self-test execution status:      (   0) The previous self-test routine completed
                                        without error or no self-test has ever
                                        been run.
Total time to complete Offline
data collection:                (    0) seconds.
Offline data collection
capabilities:                    (0x79) SMART execute Offline immediate.
                                        No Auto Offline data collection support.
                                        Suspend Offline collection upon new
                                        command.
                                        Offline surface scan supported.
                                        Self-test supported.
                                        Conveyance Self-test supported.
                                        Selective Self-test supported.
SMART capabilities:            (0x0003) Saves SMART data before entering
                                        power-saving mode.
                                        Supports SMART auto save timer.
Error logging capability:        (0x01) Error logging supported.
                                        General Purpose Logging supported.
Short self-test routine
recommended polling time:        (   1) minutes.
Extended self-test routine
recommended polling time:        (   2) minutes.
Conveyance self-test routine
recommended polling time:        (   2) minutes.
SCT capabilities:              (0x003d) SCT Status supported.
                                        SCT Error Recovery Control supported.
                                        SCT Feature Control supported.
                                        SCT Data Table supported.

SMART Attributes Data Structure revision number: 1
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
  5 Reallocated_Sector_Ct   0x0032   100   100   000    Old_age   Always       -       4
  9 Power_On_Hours          0x0032   100   100   000    Old_age   Always       -       20335        #这里可以看出磁盘已经使用了多久来辨别是否是新的磁盘
 12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       5
170 Unknown_Attribute       0x0033   099   099   010    Pre-fail  Always       -       0
171 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       0
172 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       1
174 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       3
175 Program_Fail_Count_Chip 0x0033   100   100   010    Pre-fail  Always       -       541497756210
183 Runtime_Bad_Block       0x0032   100   100   000    Old_age   Always       -       0
184 End-to-End_Error        0x0033   100   100   090    Pre-fail  Always       -       0
187 Reported_Uncorrect      0x0032   100   100   000    Old_age   Always       -       0
190 Airflow_Temperature_Cel 0x0022   078   064   000    Old_age   Always       -       22 (Min/Max 15/36)
192 Power-Off_Retract_Count 0x0032   100   100   000    Old_age   Always       -       3
194 Temperature_Celsius     0x0022   100   100   000    Old_age   Always       -       22
197 Current_Pending_Sector  0x0012   100   100   000    Old_age   Always       -       0
199 UDMA_CRC_Error_Count    0x003e   100   100   000    Old_age   Always       -       0
225 Unknown_SSD_Attribute   0x0032   100   100   000    Old_age   Always       -       724602
226 Unknown_SSD_Attribute   0x0032   100   100   000    Old_age   Always       -       184
227 Unknown_SSD_Attribute   0x0032   100   100   000    Old_age   Always       -       48
228 Power-off_Retract_Count 0x0032   100   100   000    Old_age   Always       -       1220140
232 Available_Reservd_Space 0x0033   099   099   010    Pre-fail  Always       -       0
233 Media_Wearout_Indicator 0x0032   100   100   000    Old_age   Always       -       0
234 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       0
241 Total_LBAs_Written      0x0032   100   100   000    Old_age   Always       -       724602
242 Total_LBAs_Read         0x0032   100   100   000    Old_age   Always       -       684450
243 Unknown_Attribute       0x0032   100   100   000    Old_age   Always       -       1298946

SMART Error Log Version: 1
No Errors Logged     #如果磁盘有故障则这里会显示

SMART Self-test log structure revision number 1
No self-tests have been logged.  [To run self-tests, use: smartctl -t]


SMART Selective self-test log data structure revision number 1
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0        0  Not_testing
    2        0        0  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
Selective self-test flags (0x0):
  After scanning selected spans, do NOT read-scan remainder of disk.
If Selective self-test is pending on power-up, resume after 0 minute delay.

RAID模式查询使用以下命令
sudo smartctl -a -d megaraid,24 /dev/bus/0

通过以下命令可以将raid卡的各级别日志保存进行查看
sudo /opt/MegaRAID/MegaCli/MegaCli64 -AdpEventLog -GetEvents -warning -f raid_event_warning_YYYYMMDD -aALL
sudo /opt/MegaRAID/MegaCli/MegaCli64 -AdpEventLog -GetEvents -critical -f raid_event_critical_YYYYMMDD -aALL
sudo /opt/MegaRAID/MegaCli/MegaCli64 -AdpEventLog -GetEvents -fatal -f raid_event_fatal_YYYYMMDD -aALL
若日志中Command timeout或reset的event，轻则造成IO短暂block，重则导致主机OS hang住
```

- ceph osd下线操作
```
确认osd是否已经踢出集群（down/out）
若未踢出集群则手动操作停止osd进程并踢出集群，操作前可调整ceph集群均衡速率以减轻数据均衡时对集群读写的影响
sudo ceph tell osd.* injectargs --osd_recovery_sleep 0.1 #数值范围0~0.5 数值越大均衡速率越快，影响越大

停止osd进程
systemctl stop ceph-osd@571

清理挂载点
sudo umount /var/lib/ceph/osd/ceph-571

等待超时自动out，或手动out
ceph osd out 571

若立即更换新盘则无需删除crushmap中osd信息

12版本以后还需清理lvm
使用lvremove/vgremove/pvremove清理相关信息

检查raid卡cache
sudo /opt/MegaRAID/MegaCli/MegaCli64  -GetPreservedCacheList -aALL
Adapter 0: No Virtual Drive has Preserved Cache Data.
Exit Code: 0x00

若存在cache则根据cache信息清理raid cache
sudo /opt/MegaRAID/MegaCli/MegaCli64 -DiscardPreservedCache -L1 -a0

```

