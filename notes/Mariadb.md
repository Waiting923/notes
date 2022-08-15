# Mariadb
## 集群查询
```
最大连接数
$ show variables like '%max_connections%';

当前连接数
$ show status like 'Threads%';

查看集群状态
$ show status like 'wsrep%';

主备集群
$ show slave status;
$ show master status;
```
---
## 修改最大链接数以及打开文件句柄数 
1. 确认本机limits已经修改
```shell
$ ulimit -n
65536

$ cat /etc/security/limits.conf
*     hard    nofile  65536
*     soft    nofile  65536
*     hard    nproc   65535
*     soft    nproc   65535
* soft memlock unlimited
* hard memlock unlimited
```
2. 修改服务文件
```shell
$ vim /usr/lib/systemd/system/mariadb.service
[Service]
PrivateTmp=true
LimitNOFILE=65535
LimitNPROC=65535
```

3. 修改数据库配置文件
```shell
$ vim /etc/my.cnf
[mysqld]
mariadb-server.cnf
max_connections = 10000
open_files_limit = 1048576
innodb_open_files = 2000
  ```

4. 重新加载服务配置，逐台重启
```shell
$ systemctl daemon-reload
$ systemctl restart mariadb
```
---
## Mariadb glaera集群初始化/恢复
1. 挑选数据最新节点修改安全启动
```shell
$ cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    41a7b0a1-4ce1-11ec-a02d-5e22fbb20d1d
seqno:   -1
safe_to_bootstrap: 1
```
2. 初始化/恢复
```shell
$ galera_new_cluster

$ cat /usr/bin/galera_new_cluster
$!/bin/sh

# This file is free software; you can redistribute it and/or modify it
# under the terms of the GNU Lesser General Public License as published by
# the Free Software Foundation; either version 2.1 of the License, or
# (at your option) any later version.

if [ "${1}" = "-h" ] || [ "${1}" = "--help" ]; then
    cat <<EOF

Usage: ${0}

    The script galera_new_cluster is used to bootstrap new Galera Cluster,
    when all the nodes are down. Run galera_new_cluster on the first node only.
    On the remaining nodes simply run 'service mariadb start'.

    For more information on Galera Cluster configuration and usage see:
    https://mariadb.com/kb/en/mariadb/getting-started-with-mariadb-galera-cluster/

EOF
    exit 0
fi

systemctl set-environment _WSREP_NEW_CLUSTER='--wsrep-new-cluster' && \
    systemctl start ${1:-mariadb}

extcode=$?

systemctl set-environment _WSREP_NEW_CLUSTER=''

return $extcode
```