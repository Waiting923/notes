# Rocky8.6-zfs2.1.2-lustre2.15.1

## 系统信息
```
$ cat /etc/os-release
NAME="Rocky Linux"
VERSION="8.6 (Green Obsidian)"
ID="rocky"
ID_LIKE="rhel centos fedora"
VERSION_ID="8.6"
PLATFORM_ID="platform:el8"
PRETTY_NAME="Rocky Linux 8.6 (Green Obsidian)"
ANSI_COLOR="0;32"
CPE_NAME="cpe:/o:rocky:rocky:8:GA"
HOME_URL="https://rockylinux.org/"
BUG_REPORT_URL="https://bugs.rockylinux.org/"
ROCKY_SUPPORT_PRODUCT="Rocky Linux"
ROCKY_SUPPORT_PRODUCT_VERSION="8"
REDHAT_SUPPORT_PRODUCT="Rocky Linux"
REDHAT_SUPPORT_PRODUCT_VERSION="8"
```

## 依赖包安装
```
$ dnf -y install --enablerepo=powertools asciidoc audit-libs-devel automake bc binutils-devel bison device-mapper-devel elfutils-devel elfutils-libelf-devel expect flex gcc gcc-c++ git glib2 glib2-devel hmaccalc keyutils-libs-devel krb5-devel ksh libattr-devel libblkid-devel libselinux-devel libtool libuuid-devel libyaml-devel lsscsi make ncurses-devel net-snmp-devel net-tools newt-devel numactl-devel parted patchutils pciutils-devel perl-ExtUtils-Embed pesign python3-devel redhat-rpm-config rpm-build systemd-devel tcl tcl-devel tk tk-devel wget xmlto yum-utils zlib-devel epel-release libmount-devel libyaml-devel libnl3-devel
```

## zfs安装
```
$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/zfs-2.1.2-1.el8.x86_64.rpm

$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/zfs-dkms-2.1.2-1.el8.noarch.rpm

$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/libzfs5-2.1.2-1.el8.x86_64.rpm

$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/libzpool5-2.1.2-1.el8.x86_64.rpm

$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/libnvpair3-2.1.2-1.el8.x86_64.rpm

$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/libuutil3-2.1.2-1.el8.x86_64.rpm

$ wget https://dl.rockylinux.org/vault/rocky/8.6/BaseOS/x86_64/os/Packages/k/kernel-headers-4.18.0-372.9.1.el8.x86_64.rpm

$ wget https://dl.rockylinux.org/vault/rocky/8.6/BaseOS/x86_64/os/Packages/k/kernel-devel-4.18.0-372.9.1.el8.x86_64.rpm

$ dnf localinstall ./*

$ reboot

$ modprobe zfs
```

## lustre安装
```
$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/lustre-2.15.1-1.el8.x86_64.rpm

$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/lustre-osd-zfs-mount-2.15.1-1.el8.x86_64.rpm

$ wget https://downloads.whamcloud.com/public/lustre/lustre-2.15.1/el8.6/server/RPMS/x86_64/lustre-zfs-dkms-2.15.1-1.el8.noarch.rpm


$ rpm  -ivh --nodeps ./*
#这里安装后会报错

$ cd /usr/src/lustre-zfs-2.15.1/

$ vim dkms.conf
#REMAKE_INITRD="no

$ vim configure
zfsver=$(ls -1 /usr/src/ | grep -m1 zfs | cut -f2 -d'-')
to 
zfsver=$(ls -1 /usr/src/ | grep -m1 ^zfs | cut -f2 -d'-')

$ vim lustre-dkms_pre-build.sh
ZFS_VERSION=$(dkms status -m zfs -k $3 -a $5 | awk -F', ' '{print $2; exit 0}' | grep -v ': added$')
to
ZFS_VERSION=$(dkms status -m zfs -k $3 -a $5 | awk -F', ' '{print $1; exit 0}' | cut -f2 -d'/'| cut -f1 -d':')

$ dkms build lustre-zfs/2.15.1

$ dkms install lustre-zfs/2.15.1

$ modprobe lustre

$ dkms status
Deprecated feature: REMAKE_INITRD (/var/lib/dkms/zfs/2.1.2/source/dkms.conf)
lustre-zfs/2.15.1, 4.18.0-372.9.1.el8.x86_64, x86_64: installed
zfs/2.1.2, 4.18.0-372.9.1.el8.x86_64, x86_64: installed
```

## RDMA安装

驱动下载地址：https://network.nvidia.com/products/infiniband-drivers/linux/mlnx_ofed/
```
$ dnf -y install perl pciutils gtk2 atk cairo gcc-gfortran tcsh lsof tcl tk

$ MLNX_OFED_LINUX-4.9-6.0.6.0-rhel8.6-x86_64/mlnxofedinstall

$ ibv_devices
    device          	   node GUID
    ------          	----------------
    mlx5_bond_0     	1070fd0300142d6c
```

## lnet配置
```
$ cat /etc/modprobe.d/lustre.conf
options lnet networks=o2ib(bond0)
options ko2iblnd dev_failover=1

$ lnetctl net show
net:
    - net type: lo
      local NI(s):
        - nid: 0@lo
          status: up
    - net type: o2ib
      local NI(s):
        - nid: 10.201.0.161@o2ib
          status: up
          interfaces:
              0: bond0
```

## zfs配置
```
#MGS/MDS节点
$ zpool create -O canmount=off -o ashift=12 metapool mirror /dev/nvme0n1 /dev/nvme1n1

#OSS/OST节点
$ zpool create -O canmount=off -o ashift=12 datapool mirror /dev/nvme0n1 /dev/nvme1n1

$ zpool list
NAME       SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
metapool  6.98T   452K  6.98T        -         -     0%     0%  1.00x    ONLINE  -

$ zpool status -v
  pool: metapool
 state: ONLINE
config:

	NAME         STATE     READ WRITE CKSUM
	metapool     ONLINE       0     0     0
	  mirror-0   ONLINE       0     0     0
	    nvme0n1  ONLINE       0     0     0
	    nvme1n1  ONLINE       0     0     0

errors: No known data errors
```

## 配置hostid
```
$ hid=`[ -f /etc/hostid ] && od -An -tx /etc/hostid|sed 's/ //g'`

$ [ "$hid" = `hostid` ] || genhostid
```

## MGS/MDT初始化
```
$ mkfs.lustre --reformat --mdt --mgs --backfstype=zfs --fsname=lustre1 --mountfsoptions=acl --mgsnode=10.201.0.161@o2ib --servicenode 10.201.0.161@o2ib --index=0 metapool/mdt

   Permanent disk data:
Target:     lustre1:MDT0000
Index:      0
Lustre FS:  lustre1
Mount type: zfs
Flags:      0x1065
              (MDT MGS first_time update no_primnode )
Persistent mount opts: acl
Parameters: mgsnode=10.201.0.161@o2ib failover.node=10.201.0.161@o2ib
mkfs_cmd = zfs create -o canmount=off  metapool/mdt
  xattr=sa
  dnodesize=auto
Writing metapool/mdt properties
  lustre:mgsnode=10.201.0.161@o2ib
  lustre:failover.node=10.201.0.161@o2ib
  lustre:version=1
  lustre:flags=4197
  lustre:index=0
  lustre:fsname=lustre1
  lustre:svname=lustre1:MDT0000
  lustre:mountopts=acl

$ mkdir -p /lustre/mdt

$ mount -t lustre -o acl metapool/mdt /lustre/mdt

$ ps -elf | grep mgs

$ ps -elf | grep mdt 
```

## OSS/OST初始化
```
$ mkfs.lustre --reformat --ost --backfstype=zfs --fsname=lustre1 --mgsnode=10.201.0.161@o2ib --servicenode 10.201.0.162@o2ib --index=0 datapool/ost

   Permanent disk data:
Target:     lustre1:OST0000
Index:      0
Lustre FS:  lustre1
Mount type: zfs
Flags:      0x1062
              (OST first_time update no_primnode )
Persistent mount opts:
Parameters: mgsnode=10.201.0.161@o2ib failover.node=10.201.0.162@o2ib
mkfs_cmd = zfs create -o canmount=off  datapool/ost
  xattr=sa
  dnodesize=auto
  recordsize=1M
Writing datapool/ost properties
  lustre:mgsnode=10.201.0.161@o2ib
  lustre:failover.node=10.201.0.162@o2ib
  lustre:version=1
  lustre:flags=4194
  lustre:index=0
  lustre:fsname=lustre1
  lustre:svname=lustre1:OST0000

$ mkdir /lustre/ost -p

$ mount -t lustre datapool/ost /lustre/ost

$ ps -elf | grep ost
```

## 客户端挂载
安装lustre-client以后
```
$ mount -t lustre 10.201.0.161@o2ib:/lustre1 /mnt/lustre/
```

## 参考
https://github.com/prod-feng/Install-Lustre-2.15-with-ZFS-2.1.2-on-Rocky-Linux-8.6-

https://wiki.lustre.org/Installing_the_Lustre_Software

https://wiki.lustre.org/Lustre_with_ZFS_Install