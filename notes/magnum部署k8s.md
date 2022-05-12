# Magnum 部署 k8s
官方文档：_https://docs.openstack.org/magnum/latest/user/index.html#kubernetes_

## 环境准备
由于目前使用的环境为T版，所以镜像等版本会比较老，和新版的部署细节也会有差异。

虚拟机镜像：_https://download.fedoraproject.org/pub/alt/atomic/stable/Fedora-Atomic-27-20180419.0/CloudImages/x86_64/images/Fedora-Atomic-27-20180419.0.x86_64.qcow2_

需要外网拉取的镜像：（镜像版本根据magnum版本有变化）

以下镜像必须

_docker.io/openstackmagnum/heat-container-agent:ussuri-dev

docker.io/coredns/coredns:1.3.1

quay.io/coreos/etcd:v3.2.7

docker.io/k8scloudprovider/k8s-keystone-auth:v1.18.0

docker.io/k8scloudprovider/openstack-cloud-controller-manager:v1.18.0

gcr.io/google_containers/pause:3.1

docker.io/openstackmagnum/kubernetes-apiserver:v1.15.12

docker.io/openstackmagnum/kubernetes-controller-manager:v1.15.12

docker.io/openstackmagnum/kubernetes-kubelet:v1.15.12

docker.io/openstackmagnum/kubernetes-proxy:v1.15.12

docker.io/openstackmagnum/kubernetes-scheduler:v1.15.12_

以下镜像非必须

_k8s.gcr.io/hyperkube:v1.18.2

docker.io/grafana/grafana:5.1.5

docker.io/prom/node-exporter:latest

docker.io/prom/prometheus:latest

docker.io/traefik:v1.7.28_

_gcr.io/google_containers/
kubernetes-dashboard-amd64:v1.5.1_

_gcr.io/google_containers/metrics-server-amd64:v0.3.6_

_k8s.gcr.io/node-problem-detector:v0.6.2_

_docker.io/planetlabs/draino:abf028a_

_docker.io/openstackmagnum/cluster-autoscaler:v1.18.1_

_quay.io/calico/cni:v3.13.1_

_quay.io/calico/pod2daemon-flexvol:v3.13.1_

_quay.io/calico/kube-controllers:v3.13.1_

_quay.io/calico/node:v3.13.1_

_quay.io/coreos/flannel-cni:v0.3.0_

_quay.io/coreos/flannel:v0.12.0-amd64_

_quay.io/prometheus/alertmanager:v0.20.0_

_docker.io/squareup/ghostunnel:v1.5.2_

_docker.io/jettech/kube-webhook-certgen:v1.0.0_

_quay.io/coreos/prometheus-operator:v0.37.0_

_quay.io/coreos/configmap-reload:v0.0.1_

_quay.io/coreos/prometheus-config-reloader:v0.37.0_

_quay.io/prometheus/prometheus:v2.15.2_

_docker.io/k8scloudprovider/cinder-csi-plugin:v1.18.0_

_quay.io/k8scsi/csi-attacher:v2.0.0_

_quay.io/k8scsi/csi-provisioner:v1.4.0_

_quay.io/k8scsi/csi-snapshotter:v1.2.2_

_quay.io/k8scsi/csi-resizer:v0.3.0_

_quay.io/k8scsi/csi-node-driver-registrar:v1.1.0_


openstack环境：部署好magnum

网络：需要创建public网络（若etcd使用公网发现服务部署，public网络需要连接公网，同时magnum服务的控制节点也需要连接公网）

registry：创建一台public网络的虚拟机，使用docker启动registry，将部署k8s需要使用的镜像push到local registry，同时在该节点部署https代理，使registry对外提供https服务。

## registry部署
使用public网络创建一台虚拟机，安装docker。并启动registry容器，https代理。

```
#使用镜像启动registry(镜像自己官方拉取即可)
$ docker run -d -p 5000:5000 --restart=always --name registry -v /mnt/registry:/var/lib/registry registry.cn-hangzhou.aliyuncs.com/google_containers/registry

#给准备好的k8s部署镜像打tag(举例etcd，其他镜像也相同)
$ docker tag f55bb8b733c7 localhost:5000/openstackmagnum/etcd:v3.2.


#添加信任registry到daemon.json
{ "insecure-registries":["10.3.12.125:5000"] }
$ systemctl restart docker

#push镜像到registry
$ docker push localhost:5000/openstackmagnum/etcd:v3.2.7

#获取registry内容
$ curl -X GET http://localhost:5000/v2/_catalog

{"repositories":["coredns/coredns","coreos/etcd","google_containers/pause","k8scloudprovider/k8s-keystone-auth","k8scloudprovider/openstack-cloud-controller-manager","openstackmagnum/coredns","openstackmagnum/etcd","openstackmagnum/heat-container-agent","openstackmagnum/k8s-keystone-auth","openstackmagnum/kubernetes-apiserver","openstackmagnum/kubernetes-controller-manager","openstackmagnum/kubernetes-kubelet","openstackmagnum/kubernetes-proxy","openstackmagnum/kubernetes-scheduler","openstackmagnum/openstack-cloud-controller-manager","openstackmagnum/pause"]}

#安装好nginx，配置https代理
修改nginx.conf TLS部分配置
...
ssl_certificate "/root/cert/registry.test.tech.crt"; #事先生成的证书
ssl_certificate_key "/root/cert/registry.test.tech.key"; #事先生成的密钥
...
location / {
            proxy_set_header Host $host;
            proxy_pass http://10.3.12.125:5000/;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
...
重启生效

使用其他节点验证
$ docker pull registry.test.tech/openstackmagnum/etcd:v3.2.7
```

## magnum创建
由于需要使用local registry，所以这里的fedora镜像我自己更新了一版，在里面加上了对registry的域名解析，在/etc/docker/certs.d/中添加了自生成证书。

```
#上传fedora镜像
$ openstack image create --disk-format=qcow2 --container-format=bare --file=Fedora-Atomic-27-20180419.0.x86_64.qcow2 --property os_distro='fedora-atomic' fedora-atomic-latest

#创建虚拟机keypair

#创建magnum模板（这里若不指定etcd的discovery地址，则默认使用关方公网discovery，所以需要magnum节点通公网）
$ openstack coe cluster template create kubernetes-cluster-template-2 --image fedora-update --master-flavor 2C.4G --flavor 2C.4G --external-network public --keypair k8s.key --coe kubernetes

#创建k8s集群(这里的参数根据自己的版本以及地址修改)
$ openstack coe cluster create kubernetes-cluster --cluster-template kubernetes-cluster-template-2 --master-count 1 --node-count 1 --label kube_tag=v1.15.12,coredns_tag=1.3.1,container_infra_prefix=registry.test.tech/openstackmagnum/,etcd_tag=v3.2.7,cloud_provider_tag=v1.18.0,k8s_keystone_auth_tag=v1.18.0
```

## Troubleshooting

magnum创建k8s集群的流程：

magnum调用heat编排来进行资源调度创建，当网络，路由，安全组，虚拟机创建完成后，会通过cloud-init在fedora虚拟机中使用atomic创建heat-agent，后续k8s集群的部署会通过heat-agent调用脚本完成。

资源创建阶段故障排查：
```
查询资源创建卡在哪个阶段
$ openstack stack list
$ openstack stack resource list
$ openstack stack resource show 
```

集群部署阶段排查：
```
使用keypair登陆虚拟机（用户fedora）
查询以下两个日志
cloud-init阶段
/var/log/cloud-init-output.log
heat编排阶段
/var/log/heat-config/xxxxxxx.log

atomic排查
$ sudo atomic containers list
$ sudo atomic images list
atomic创建的服务通过systemd管理
使用journactl查询对应服务日志
systemctl查询对应服务状态
```

