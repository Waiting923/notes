# 日志环境k8s部署

## 环境准备
- master节点*3
- worker节点*3
- istio节点*1
- 节点通公网


## sealos安装
apt 源安装
```
$ echo "deb [trusted=yes] https://apt.fury.io/labring/ /" | sudo tee /etc/apt/sources.list.d/labring.list
$ sudo apt update
$ sudo apt install sealos
```

## sealos生成配置
```
$ sealos gen labring/kubernetes:v1.25.0 labring/helm:v3.8.2 labring/calico:v3.24.1 --masters xxx.xxx.xxx.xxx,xxx.xxx.xxx.xxx,xxx.xxx.xxx.xxx --nodes xxx.xxx.xxx.xxx,xxx.xxx.xxx.xxx,xxx.xxx.xxx.xxx,xxx.xxx.xxx.xxx >
 Clusterfile
```

## 修改cluster-cidr/service-cluster-ip-range
```
$ vim Clusterfile
#修改Networking字段
Networking:
  PodSubnet: xxx.xxx.xxx.xxx/xx
  ServiceSubnet: xxx.xxx.xxx.xxx/xx
#新增calico配置
---
apiVersion: apps.sealos.io/v1beta1
kind: Config
metadata:
  name: calico
spec:
  path: charts/calico/values.yaml
  strategy: merge
  data: |
    installation:
      enabled: true
      kubernetesProvider: ""
      calicoNetwork:
        ipPools:
        - blockSize: 26
          cidr: xxx.xxx.xxx.xxx/xx #与PodSubnet保持一致
          encapsulation: IPIP
          natOutgoing: Enabled
          nodeSelector: all()
        nodeAddressAutodetectionV4:
          interface: "eth0" #多网卡需要指定calico网卡
```
k8s v1.25不支持以下参数需要删除Clusterfile中所有对应参数
[PR](https://github.com/labring/sealos/pull/2340)
```
feature-gates: TTLAfterFinished=true,EphemeralContainers=true
experimental-cluster-signing-duration: 876000h
```

## sealos部署
```
$ sealos apply -f Clusterfile
#执行以下命令重新部署
$ sealos reset
```

## calico参数调整
sealos使用tigera-operator部署calico，修改配置需要编辑Installation
```
$ kubectl  get Installation
NAME      AGE
default   16h
$ kubectl edit Installation default
```

## 所有节点containerd添加私有仓库
sealos部署的containerd config_path应该使用certs.d
```
$ vim /etc/containerd/config.toml
[plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
```
通过脚本添加仓库配置
```
$ vim gen-containerd-hosts.sh
#!/bin/bash

registry=${1:-${REGISTRY}}

registry_server=${REGISTRY_SERVER:-${registry}}

if [[ -z "${registry}" ]]; then
    echo empty registry domain
    exit 1
fi

conf_dir=${CONTAINED_CONFIG_DIR:-/etc/containerd/certs.d}

mkdir -p "$conf_dir/${registry}"

cat << EOF > "$conf_dir/${registry}"/hosts.toml
server = "http://${registry_server}"
[host."http://${registry_server}"]
  capabilities = ["pull", "resolve"]
  skip_verify = true
EOF

chmod 755 "$conf_dir/${registry}"/hosts.toml

./gen-containerd-hosts.sh harbor.xxx.xxx
```

## coredns添加上游dns
```
$ kubectl -n kube-system edit cm coredns
xxx.cn:53 {
        errors
        cache 30
        forward . xxx.xxx.xxx.xxx
        reload
    }
```

## 节点添加label
```
$ kubectl label node k8s-istio-1 kubernetes.io/role=istio
$ kubectl label node k8s-worker-1 kubernetes.io/role=node
$ kubectl label node k8s-worker-2 kubernetes.io/role=node
$ kubectl label node k8s-worker-3 kubernetes.io/role=node
$ kubectl label node k8s-worker-1 role=worker
$ kubectl label node k8s-worker-2 role=worker
$ kubectl label node k8s-worker-3 role=worker
```

## 生成admin token
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kube-system
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin
subjects:
  - kind: ServiceAccount
    name: admin
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: admin
  namespace: kube-system
  annotations:
    kubernetes.io/service-account.name: "admin"

$ kubectl -n kube-system describe secret admin
```

## istio部署
```
$ wget https://github.com/istio/istio/releases/download/1.16.1/istio-1.16.1-linux-amd64.tar.gz

$ vim istio-1.16.1/manifests/charts/gateways/istio-ingress/values.yaml
nodeSelector: {kubernetes.io/role: istio}

$ vim istio-1.16.1/manifests/charts/gateways/istio-ingress/templates/deployment.yaml
spec:
  template:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet

$ vim istio-1.16.1manifests/charts/istio-control/istio-discovery/values.yaml
nodeSelector: {kubernetes.io/role: istio}

$ istio-1.16.1/bin/istioctl install --manifests=manifests/

$ istio-1.16.1/bin/istioctl uninstall --purge
```
## 修改ingress-gw external-ip
```
$ kubectl  -n istio-system edit svc istio-ingressgateway
spec:
  externalIPs:
  - xxx.xxx.xxx.xxx
```
## 根据需求配置cert/gw/policy/vs