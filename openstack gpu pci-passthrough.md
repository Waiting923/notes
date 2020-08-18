# Openstack gpu显卡透传
## 显卡信息
> Nvidia Quadro RTX6000  
  Nvidia Tesla V100S
## 物理节点需求
> CPU需要支持VT-d,VT-x （VT-d:输入/输出虚拟化）  
  主板支持IOMMU GROUP
## 透传参考链接
- [PCI passthrough via OVMF](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF_(%E7%AE%80%E4%BD%93%E4%B8%AD%E6%96%87)#%E5%8F%A6%E8%AF%B7%E5%8F%82%E9%98%85)  
- [Attaching physical PCI devices to guests](https://docs.openstack.org/nova/pike/admin/pci-passthrough.html)
## 问题
- [ 描述 ] 由于显卡上带有type-c接口，通过正常vfio-pci的方式进行设备隔离，重启物理节点后，type-c端口重新被物理机的内核usb3.0 xhci_hcd所管理，导致透传失败。而xhci_hcd被编译在内核中，无法通过blacklist方式来将module禁用。  
![type-c.png](https://github.com/shutupandrun/picture/blob/master/type-c.png?raw=true)

- [ 解决方式 ] 将type-c接口通过pci-stub方式进行隔离  
![pci-stub.png](https://github.com/shutupandrun/picture/blob/master/pci-stub.png)
