# Openstack train 离线安装readme
## 前置条件
- 操作系统版本Centos7.7-1908-minimal（其他版本未进行测试）
- 系统安装完成
- 主机名配置完成
- 网卡规划配置完成
- deploy节点到其他节点免密配置完成

## 使用方法
- 将workspace-xxxxxx.tar.gz上传至deploy节点/root路径
- 解压该文件
```
tar -xvzf workspace-xxxxxx.tar.gz
```
- 执行命令
```
cd /root/workspace
./createrepo.sh
```
- 编辑node
```
vim /root/workspace/node
[all:vars]
deploy_ip=控制节点控制网络ip地址
[deploy]
部署节点主机名

[node]
其他节点主机名
```
- 执行命令
```
ansible-playbook -i node setup.yml
```
- 编辑multinode(根据规划修改)
```
vim /root/workspace/multinode
[control]
控制节点主机名
[network]
网络节点主机名
[compute]
计算节点主机名
[storage]
ceph osd 节点主机名
```
- 编辑globals.yml
```
vim /etc/kolla/globals.yml
kolla_internal_vip_address: "{{ vip_address }}" #vip_address替换为openstack集群vip
docker_registry: "{{ deploy_ip }}:{{ registry_port }}" #deploy_ip替换为部署节点管理网ip地址，registry_port替换为4000
network_interface: "{{  mt_port }}" #mt_port替换为所有节点管理网口名称
tunnel_interface: "{{ vx_port }}" #vx_port替换为所有节点vxlan网口名称
neutron_external_interface: "{{ bz_port }}" #bz_port替换为所有节点业务网口名称（vlan/flat）
```
- ceph osd 磁盘打标签
```
cd /root/workspace
vim ./ceph/osd_tag.sh #根据真实osd磁盘盘符进行脚本修改
ansible -i multinode -m copy -a "src=/root/workspace/ceph/osd_tag.sh dest=/root" storage #拷贝脚本到目标节点
ansible -i multinode -m shell -a "bash /root/osd_tag.sh" storage
#如果需要磁盘分区清理则执行以下命令
vim ./ceph/osd_clean.sh #根据真实osd磁盘盘符进行脚本修改
ansible -i multinode -m copy -a "src=/root/workspace/ceph/osd_clean.sh dest=/root" storage #拷贝脚本到目标节点
ansible -i multinode -m shell -a "bash /root/osd_clean.sh" storage
```

- 部署预检查
```
kolla-ansible -i multinode prechecks
```

- 部署
```
kolla-ansible -i multinode deploy
```

- 生成openrc
```
kolla-ansible -i multinode post-deploy
```

注：setup.yml 单步执行在命令后加 -t {tags}
- __tags:__
- system (关闭selinux、firewalld)
- copy_repo(生成所有节点yum.repo)
- yum_install (安装依赖包)
- deploy_pip_install(deploy节点三方库安装)
- node_pip_install(其他节点三方库安装)
- docker(配置并启动所有节点docker)
- registry(docker registry)
- kolla_config(kolla-ansible初始化)
