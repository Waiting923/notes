# Openstack gpu显卡透传
## 显卡信息
> Nvidia Quadro RTX6000  
  Nvidia Tesla V100S  
  3080  
  3090  
  A40
## 物理节点需求
> CPU需要支持VT-d,VT-x （VT-d:输入/输出虚拟化）  
  主板支持IOMMU GROUP(bios开启)

#注： 超微机器开启iommu同时，开启网卡的pci-sriov会出现问题。
## 节点初始化
[gpu-init.yaml](https://github.com/Riverdd/scripts/blob/master/gpu-init.yaml)
- 开启iommu(intel cpu: intel_iommu, amd cpu: amd_iommu)
```
$ vim /etc/default/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto amd_iommu=on iommu=pt rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet"
GRUB_DISABLE_RECOVERY="true"

$ grub2-mkconfig -o /boot/grub2/grub.cfg
```
- 关闭nouveau模块
```
$ vim /etc/modprobe.d/blacklist.conf
blacklist nouveau
options nouveau modset=0

#备份原initramfs镜像
$ mv /boot/initramfs-$(uname -r).img /boot/initramfs-$(uname -r)-nouveau.img

#生成新initramfs镜像
$ dracut /boot/initramfs-$(uname -r).img $(uname -r)

#Ubuntu命令
$ apt install initramfs-tools
update-initramfs -u
```
- 加载vfio模块
```
$ vim /etc/modules-load.d/vfio.conf
vfio
vfio-pci
vfio_iommu_type1
br_netfilter
```
- 显卡绑定vfio(自动)
```
$ vim /etc/modprobe.d/vfio.conf
options vfio-pci ids=10de:2235
```
- 重启后检查生效
```
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-3.10.0-1160.el7.x86_64 root=/dev/mapper/centos-root ro crashkernel=auto amd_iommu=on iommu=pt rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet

$ dmesg | grep IOMMU (AMD)
[   10.021227] AMD-Vi: IOMMU performance counters supported
[   10.021290] AMD-Vi: IOMMU performance counters supported
[   10.021315] AMD-Vi: IOMMU performance counters supported
[   10.021341] AMD-Vi: IOMMU performance counters supported
[   10.021391] AMD-Vi: IOMMU performance counters supported
[   10.021421] AMD-Vi: IOMMU performance counters supported
[   10.021453] AMD-Vi: IOMMU performance counters supported
[   10.021481] AMD-Vi: IOMMU performance counters supported
[   10.097357] AMD-Vi: Found IOMMU at 0000:60:00.2 cap 0x40
[   10.097363] AMD-Vi: Found IOMMU at 0000:40:00.2 cap 0x40
[   10.097366] AMD-Vi: Found IOMMU at 0000:20:00.2 cap 0x40
[   10.097370] AMD-Vi: Found IOMMU at 0000:00:00.2 cap 0x40
[   10.097373] AMD-Vi: Found IOMMU at 0000:e0:00.2 cap 0x40
[   10.097377] AMD-Vi: Found IOMMU at 0000:c0:00.2 cap 0x40
[   10.097380] AMD-Vi: Found IOMMU at 0000:a0:00.2 cap 0x40
[   10.097384] AMD-Vi: Found IOMMU at 0000:80:00.2 cap 0x40

$ dmesg | grep IOMMU (Intel)
[    0.000000] DMAR: IOMMU enabled
[    0.323374] DMAR-IR: IOAPIC id 8 under DRHD base  0x9b7fc000 IOMMU 9

$ lsmod | grep nouveau

$ lsmod | grep vfio
vfio_pci               41993  0
vfio_iommu_type1       22440  0
vfio                   32657  2 vfio_iommu_type1,vfio_pci
irqbypass              13503  2 kvm,vfio_pci

$ lspci -nnk -d 10de:2235
21:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:2235] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:145a]
	Kernel modules: nouveau
```
若重启后未生效可以使用手动绑定方式进行测试
- 显卡绑定vfio(手动)
```
$ lspci -nnv | grep -i nvidia
21:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:2235] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:145a]

$ lspci -nnk -d 10de:2235
21:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:2235] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:145a]
	Kernel modules: nouveau

#若检查后发现原显卡仍绑定在原nouveau驱动上则进行解绑，若没有绑定则无需操作。(若显卡自带声卡，注意声卡也同样需要透传)
$ echo 0000:21:00.0 > /sys/bus/pci/devices/0000:21:00.0/driver/unbind

$ echo 10de 2235 > /sys/bus/pci/drivers/vfio-pci/new_id

$ ls /sys/bus/pci/drivers/vfio-pci/
0000:21:00.0  bind  module  new_id  remove_id  uevent  unbind

$ lspci -nnk -d 10de:2235
21:00.0 3D controller [0302]: NVIDIA Corporation Device [10de:2235] (rev a1)
	Subsystem: NVIDIA Corporation Device [10de:145a]
	Kernel driver in use: vfio-pci
	Kernel modules: nouveau
```

- 绑定成功后开始配置openstack pci-passthrough

计算节点
```
#nova-compute
nova.conf
[pci]
alias = {"name": "gpuA40", "vendor_id": "10de", "product_id": "2235"}
passthrough_whitelist = {"vendor_id": "10de", "product_id": "2235"}
```
控制节点
```
#nova-api, nova-scheduler, nova-conductor
[pci]
alias = {"name": "gpuA40", "vendor_id": "10de", "product_id": "2235"}
```
flavor元数据
```
--property pci_passthrough:alias="gpuA40:1"
```
## 透传参考链接
- [PCI passthrough via OVMF](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E5%8F%A6%E8%AF%B7%E5%8F%82%E9%98%85)  
- [Attaching physical PCI devices to guests](https://docs.openstack.org/nova/pike/admin/pci-passthrough.html)
## 问题
- [ 描述 ] 由于显卡上带有type-c接口，通过正常vfio-pci的方式进行设备隔离，重启物理节点后，type-c端口重新被物理机的内核usb3.0 xhci_hcd所管理，导致透传失败。而xhci_hcd被编译在内核中，无法通过blacklist方式来将module禁用。  
![type-c.png](https://github.com/Riverdd/picture/blob/master/type-c.png?raw=true)

- [ 解决方式 ] 将type-c接口通过pci-stub方式进行隔离  
![pci-stub.png](https://github.com/Riverdd/picture/blob/master/pci-stub.png)

- 多个pci设备同时透传到同一台虚拟机时，flavor可以如下配置
```
openstack flavor set --property pci_passthrough:alias="nvme155:1,gpu3080:1" flavor3080
```
- 可以手动kvm挂载gpu设备，但是openstack-scheduler pcipassthrough失败
```
#在对应节点创建不带gpu的虚拟机后
$ vim a40.xml
<hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio' />
    <source>
        <address domain='0x0000' bus='0x21' solt='0x00' function='0x0'  /> #bus对应lspci的pci地址
    </source>
</hostdev>

$ virsh attach-device instance-000044e2 a40.xml
```
查询数据库发现识别device_type为type-PF
```
MariaDB [(none)]> use nova;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
MariaDB [nova]> select * from pci_devices where product_id='2235'\G
*************************** 1. row ***************************
     created_at: 2022-08-29 02:14:35
     updated_at: 2022-08-29 07:50:58
     deleted_at: NULL
        deleted: 0
             id: 141
compute_node_id: 72
        address: 0000:21:00.0
     product_id: 2235
      vendor_id: 10de
       dev_type: type-PF
         dev_id: pci_0000_21_00_0
          label: label_10de_2235
         status: available
     extra_info: {}
  instance_uuid: NULL
     request_id: NULL
      numa_node: 0
    parent_addr: NULL
           uuid: 8c505d6e-f52c-40b7-b64f-9c675bd100a5
```
修改对应节点pci配置为"device_type":"type-PF"
```
alias = {"name": "gpua40", "vendor_id": "10de", "product_id": "2235", "device_type":"type-PF"}
```
- 查询pci设备iommu_group脚本
```
#!/bin/bash
shopt -s nullglob
for d in /sys/kernel/iommu_groups/*/devices/*; do 
    n=${d#*/iommu_groups/*}; n=${n%%/*}
    printf 'IOMMU Group %s ' "$n"
    lspci -nns "${d##*/}"
done;
```