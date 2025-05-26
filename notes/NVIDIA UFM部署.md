# NVIDIA UFM 部署
## 官方文档
_https://docs.nvidia.com/networking/display/ufmenterpriseqsgv6141_

## 前置条件

1. NVIDIA UFM license
2. 安装mellanox IB驱动，正确识别网卡，统一名称
3. 准备drbd数据盘
4. UFM docker image 和 UFM HA cluster软件包
5. 配置时间同步，确认时区为UTC

## 安装依赖包
```
$ apt -y install pacemaker pcs drbd-utils resource-agents-extra docker-ce
```

## 初始化配置
```
$ mkdir -p /opt/ufm/files

$ mkdir -p /tmp/license_file

$ cp mlnx-ufm-R3-A-0000543982_1-MLNX-20231014023952.lic /tmp/license_file/

$ docker run -it --name=ufm_installer --rm -v /var/run/docker.sock:/var/run/docker.sock -v
 /etc/systemd/system/:/etc/systemd_files/ -v /opt/ufm/files/:/installation/ufm_files/ -v /tmp/l
icense_file/:/installation/ufm_licenses/ mellanox/ufm-enterprise:latest --install

#生成的配置文件在/opt/ufm/files/下
```

## 修改配置文件
```
#gv.cfg
fabric_interface = ibs5
ufma_interfaces = team1.13
mgmt_interface = team1.13
global_m_key_seed = 0x0000000000000126

#opensm.conf
guid 0xa088c20300790f24 #对应ibstat中ib口的Node GUID
m_key 0x0000000000000126

#如果为原opensm节点替换，需要将以下文件丢入conf/opensm路径
/var/cache/opensm/guid2groupid
/var/cache/opensm/guid2lid
/var/cache/opensm/guid2mkey

#partitions.conf按照正常需求修改
```

## drbd磁盘格式化
```
mkfs.ext4 /dev/sdb
```

## 安装UFM HA
```
$ cd ufm_ha_5.7.0-6

#主备节点都执行
$ ./install.sh -l /opt/ufm/files/ -d /dev/sdb -p enterprise

$ systemctl daemon-reload

#备节点 -e对端，-l本端，-i vip
$ ufm_ha_cluster config -r standby -e 10.4.53.11 -l 10.4.53.12 -i 10.4.53.10 --enable-single-link

#主节点
$ ufm_ha_cluster config -r master -e 10.4.53.12 -l 10.4.53.11 -i 10.4.53.10 --enable-single-link

$drbd status

#若drbd未自动连接同步则手动同步
$drbdadm connect ha_data

#配置虚拟网卡以及mac
$ ip link add eth10 type dummy
$ ip link add eth20 type dummy
$ ip link set dev eth10 address e8:eb:d3:bf:5e:1c
$ ip link set dev eth20 address e8:eb:d3:73:7e:64

#等待同步完成后启动集群
$ ufm_ha_cluster start
$ ufm_ha_cluster status

#确认opensm状态
$ sminfo -y 0x126
```

## 修改密码
```
$ docker exec -it -u root ufm bash
$ htpasswd -B /opt/ufm/conf/ufm.passwd admin
$ ufm_ha_cluster stop
$ ufm_ha_cluster start
```

## 登陆界面
_https://10.4.53.10/ufm_


## nvidia各驱动下载链接
_https://developer.download.nvidia.cn/compute/cuda/repos/ubuntu2004/x86_64/_