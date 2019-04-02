# 基于kolla-ansible 的 Q 版本openstack搭建
## 环境准备
- 首先准备三台8C/24G,centos7虚拟机(根磁盘100G/台)作为物理节点如图所示
![image](https://raw.githubusercontent.com/shutupandrun/picture/master/xunijizhunbei.png)
- 每台机器准备两张网卡，一张用来做管理网络/存储网络（50），另外一张来做业务网络（60）。
- 在第一台节点添加主机名解析
```
[root@river-1 ~]# vim /etc/hosts

[root@river-1 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.16.50.12 river-1
172.16.50.158 river-2
172.16.50.138 river-3
```
- 修改业务网卡为无ip(3个节点都要更改，这里我以一个节点为例,如果有能力也可以在准备完ansible以后来做这一步)
```
[root@river-1 ~]# cd /etc/sysconfig/network-scripts/
[root@river-1 network-scripts]# cp ifcfg-eth0 ifcfg-eth1
[root@river-1 network-scripts]# vim ifcfg-eth1
[root@river-1 network-scripts]# cat ifcfg-eth1
# Created by cloud-init on instance boot automatically, do not edit.
#
BOOTPROTO=none
DEVICE=eth1
#HWADDR=fa:16:3e:cb:9f:bb
ONBOOT=yes
TYPE=Ethernet
USERCTL=no
[root@river-1 network-scripts]# ifup eth1
[root@river-1 network-scripts]# ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:cb:9f:bb brd ff:ff:ff:ff:ff:ff
    inet 172.16.50.12/24 brd 172.16.50.255 scope global dynamic eth0
       valid_lft 69477sec preferred_lft 69477sec
    inet6 fe80::f816:3eff:fecb:9fbb/64 scope link
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether fa:16:3e:79:eb:8b brd ff:ff:ff:ff:ff:ff
    inet6 fe80::f816:3eff:fe79:eb8b/64 scope link tentative
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN
    link/ether 02:42:b8:2d:13:4c brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
```

- 安装相关环境包
```
[root@river-1 ~]# yum install epel-release

[root@river-1 ~]# yum install python-pip

[root@river-1 ~]# pip install -U pip

[root@river-1 ~]# yum install python-devel libffi-devel gcc openssl-devel
```
---
## ansible准备

- 安装ansible
```
[root@river-1 ~]# yum install ansible
```
- 升级ansible到最新版
```
[root@river-1 ~]# pip install -U ansible
```
- 修改ansible配置文件调优

```
[root@river-1 ~]# vim /etc/ansible/ansible.cfg
#default组加一行 forks=100
```

- 节点之间免密钥登录配置
```
[root@river-1 ~]# cd .ssh
[root@river-1 .ssh]# ssh-keygen  #一路回车就可以
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:sctCvGxGakKzXfbqh9F3v6hifIFxBS05/2SqgApIy4A root@river-1
The key's randomart image is:
+---[RSA 2048]----+
|          .+     |
|          + o    |
|.       .  =     |
|E.   .  .o. . o  |
|+.+   *oS+   =   |
|.+.+ Bo++.o o .  |
|  o.+.*++o + .   |
|   o.o.o= o  ..  |
|     .oo o... .. |
+----[SHA256]-----+

[root@river-1 .ssh]# ssh-copy-id root@172.16.50.12  #注意本机也要免密钥登录

[root@river-1 .ssh]# ssh-copy-id root@172.16.50.158

[root@river-1 .ssh]# ssh-copy-id root@172.16.50.138

[root@river-1 ~]# ssh river-3  #每个节点都登录一次输入yes

[root@river-1 ~]# ssh river-2

[root@river-1 ~]# ssh river-1
```
- 对ansible进行简单测试
```
[root@river-1 ~]# vim node
[root@river-1 ~]# cat node
[node]
river-1
river-2
river-3

[root@river-1 ~]# ansible -i node -m ping all  #结果显示success并且为绿色则通过
river-2 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
river-3 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
river-1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
- 用ansible批量调整系统参数

```
[root@river-1 ~]# ansible -i node -m shell -a "setenforce 0 ; getenforce" all
river-3 | CHANGED | rc=0 >>
Permissive

river-1 | CHANGED | rc=0 >>
Permissive

river-2 | CHANGED | rc=0 >>
Permissive

[root@river-1 ~]# ansible -i node -m shell -a "sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config" all

[root@river-1 ~]# ansible -i node -m shell -a "systemctl stop firewalld && systemctl disable firewalld" all   #如果返回红色报错可能firewalld服务本来就没有开启
```
- 安装所需包
```
[root@river-1 ~]# ansible -i node -m shell -a "yum -y install epel-release python-pip python-devel libffi-devel gcc openssl-devel git wget vim net-tools" all

[root@river-1 ~]# ansible -i node -m shell -a "pip install -U pip" all
```
---
## docker准备
- 获得docker源
```
[root@river-1 ~]# ansible -i node -m shell -a "wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo" all
```
- 安装docker
```
[root@river-1 ~]# ansible -i node -m shell -a "yum -y install docker-ce" all
```
- 生成docker配置文件
```
[root@river-1 ~]# ansible -i node -m shell -a "mkdir /etc/systemd/system/docker.service.d" all

[root@river-1 ~]# cd /etc/systemd/system/docker.service.d/
[root@river-1 docker.service.d]# vim kolla.conf
[root@river-1 docker.service.d]# cat kolla.conf
[Service]
MountFlags=shared
```
- copy到另外两台节点
```
[root@river-1 ~]# ansible -i node -m copy -a "src=/etc/systemd/system/docker.service.d/kolla.conf dest=/etc/systemd/system/docker.service.d/" all
```
- 修改启动项
```
[root@river-1 ~]# vim /usr/lib/systemd/system/docker.service

#修改这一条 ExecStart=/usr/bin/dockerd --registry-mirror=http://f2d6cb40.m.daocloud.io --storage-driver=overlay2

[root@river-1 ~]# ansible -i node -m copy -a "src=/usr/lib/systemd/system/docker.service dest=/usr/lib/systemd/system/docker.service" all
```
- 启动并检查一下docker状态
```
[root@river-1 ~]# ansible -i node -m shell -a "systemctl daemon-reload && systemctl start docker && systemctl enable docker &&systemctl status docker" all

[root@river-1 ~]# ansible -i node -m shell -a "docker version" all  #检查docker版本
```
---
## kolla-ansible准备
- 在第一个节点下载kolla-ansible代码

```
[root@river-1 ~]# git clone https://github.com/openstack/kolla-ansible -b stable/queens
```
- 将代码内的配置模版copy到相应路径

```
[root@river-1 kolla-ansible]# cd kolla-ansible/
[root@river-1 kolla-ansible]# cp -r etc/kolla/ /etc/kolla/
```
- 进行安装
```
[root@river-1 kolla-ansible]#  pip install . -i https://pypi.tuna.tsinghua.edu.cn/simple
```
---
- [安装过程中可能出现以下问题 ]
```
1. oslo-config 6.8.1 has requirement PyYAML>=3.12, but you'll have pyyaml 3.10 which is incompatible.
```

```
2. Cannot uninstall 'requests'. It is a distutils installed project and thus we cannot accurately determine which files belong to it which would lead to only a partial uninstall.
```

- [解决办法如下 ]

```
1. [root@river-1 kolla-ansible]# find / -name PyYAML*
/usr/lib64/python2.7/site-packages/PyYAML-3.10-py2.7.egg-info
/usr/share/doc/PyYAML-3.10

[root@river-1 kolla-ansible]# mv /usr/lib64/python2.7/site-packages/PyYAML-3.10-py2.7.egg-info /root/
```

```
2. [root@river-1 kolla-ansible]# find / -name requests*
/usr/lib/python2.7/site-packages/requests-2.6.0-py2.7.egg-info
/usr/lib/python2.7/site-packages/requests
/usr/lib/python2.7/site-packages/pip/_vendor/requests

[root@river-1 kolla-ansible]# mv /usr/lib/python2.7/site-packages/requests-2.6.0-py2.7.egg-info /root/
```
---
## Ceph准备
- 首先准备6块磁盘作为ceph-osd，每台机器挂载两块
![image](https://raw.githubusercontent.com/shutupandrun/picture/master/cipan.png)
- 为3台节点上挂载的6个磁盘打标签（这里我只以一个节点为例，另外两台也要做）
```
[root@river-1 ~]# for i in b c;do parted /dev/vd$i -s -- mklabel gpt mkpart KOLLA_CEPH_OSD_BOOTSTRAP 1 -1;done
[root@river-1 ~]# for i in b c;do parted /dev/vd$i print;done
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                      标志
 1      1049kB  10.7GB  10.7GB               KOLLA_CEPH_OSD_BOOTSTRAP

Model: Virtio Block Device (virtblk)
Disk /dev/vdc: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name                      标志
 1      1049kB  10.7GB  10.7GB               KOLLA_CEPH_OSD_BOOTSTRAP
```
---
## 部署配置文件准备
- 准备multinode文件
```
[root@river-1 kolla]# cd
[root@river-1 ~]# cp kolla-ansible/ansible/inventory/* .
[root@river-1 ~]# vim multinode
```
- [具体multinode文件点击github链接查看]
[link](https://github.com/shutupandrun/picture/blob/master/multinode)

- 准备globals.yml文件
```
[root@river-1 ~]# vim /etc/kolla/globals.yml
```
- [具体global.yml文件点击github链接查看]
[link](https://github.com/shutupandrun/picture/blob/master/globals.yml)

- 密码初始化生成
```
[root@river-1 ~]# kolla-genpwd
[root@river-1 ~]# vim /etc/kolla/passwords.yml  #文件内有内容则成功
```

- 进行部署检查
```
[root@river-1 ~]# kolla-ansible -i multinode prechecks  #无红色报错则全部通过
```

- docker镜像拉取(先拉取镜像可以节省部署时间)
```
[root@river-1 ~]# kolla-ansible -i multinode pull
```
- docker镜像查看
```
[root@river-1 ~]# docker images
```
---
## 开始部署

```
[root@river-1 kolla]# kolla-ansible -i multinode deploy

[root@river-1 kolla]# kolla-ansible -i multinode post-deploy
```

- [部署过程中遇到过ceph问题如下]

```
TASK [ceph : Fetching Ceph keyrings] **********************************************************************************************************
fatal: [river-1]: FAILED! => {"msg": "The conditional check '(ceph_files_json.stdout | from_json).changed' failed. The error was: No JSON object could be decoded"}
```
- [解决办法如下]

```
#可能原因是ceph的配置没有清空
[root@river-1 kolla]# docker volume ls
DRIVER              VOLUME NAME
local               aodh
local               ceilometer
local               ceph_mon
local               ceph_mon_config
local               cinder
local               elasticsearch
local               glance
local               gnocchi
local               grafana
local               haproxy_socket
local               kolla_logs
local               libvirtd
local               mariadb
local               neutron_metadata_socket
local               nova_compute
local               nova_libvirt
local               nova_libvirt_qemu
local               openvswitch_db
local               rabbitmq
[root@river-1 kolla]# docker volume rm ceph_mon ceph_mon_conf
```
---
## Zabbix准备
- 首先把代码替换到对应的路径

```
kolla-zabbix.yml放到/usr/share/kolla-ansible/ansible
kolla-zabbix文件夹放到/usr/share/kolla-ansible/ansible/roles
```
- 通过其他registry下载zabbix镜像或者手动下载zabbix镜像
```
#每台节点上都要有这三个zabbix的docker镜像
centos-source-zabbix-agent
centos-source-zabbix-web
centos-source-zabbix-server
```
- 将准备好的镜像重新打tag
```
[root@river-2 docker]# docker tag 172.16.143.11:4000/99cloud/centos-source-zabbix-agent:animbus-6.5.0 kolla/centos-source-zabbix-agent:queens

[root@river-2 docker]# docker tag 172.16.143.11:4000/99cloud/centos-source-zabbix-web:animbus-6.5.0 kolla/centos-source-zabbix-web:queens

[root@river-2 docker]# docker tag 172.16.143.11:4000/99cloud/centos-source-zabbix-server:animbus-6.5.0 kolla/centos-source-zabbix-server:queens
```
- 安装grafana的zabbix-plugin(每个有grafana容器的节点都要进行以下操作，这里我以几个节点为例)
```
[root@river-2 ~]# docker exec -it -u root grafana bash
(grafana)[root@river-2 grafana]# grafana-cli plugins install alexanderzobnin-zabbix-app
```
- 然后在界面上把zabbix-plugin enable
![image](https://raw.githubusercontent.com/shutupandrun/picture/master/plugin.png)
![image](https://raw.githubusercontent.com/shutupandrun/picture/master/enable.png)
- 开始部署zabbix
```
[root@river-1 ~]# kolla-ansible -i multinode deploy -p /usr/share/kolla-ansible/ansible/kolla-zabbix.yml
```
- TASK [kolla-zabbix : import zabbix database schema]可能会运行很长时间
- 部署完成
