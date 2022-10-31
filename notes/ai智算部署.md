- Rancher
  - CMP: https://10.4.1.151/dashboard/home
    - admin / Yovole.com
  - AI: https://10.4.62.152/dashboard/home
    - admin / Yovole.com.ai


mount -t ceph 100.71.50.151:6789,100.71.50.152:6789,100.71.50.153:6789:/dataset /dataset -o rw,name=forcestack

100.71.50.151:6789,100.71.50.152:6789,100.71.50.153:6789:/ /cephfs	ceph	rw,name=forcestack,noatime,_netdev 0 0



docker run -d --privileged --restart=unless-stopped --net=host -v /etc/kubernetes:/etc/kubernetes -v /var/run:/var/run  harbor-ai.yovole.io/rancher/rancher-agent:v2.6.4 --server https://rancher-ai.yovole.io --token hrljdkcf26jgf9rz8mgkfshkldkrcxb9sfp6mmjcf7zz7jpzqdhfxt --ca-checksum 54f701e380a83ad792884f84b038f4172f51759eadff4387c6fa2a688811c84b --worker --label node.yovole.com/cache-type=nvme --label node.yovole.com/model=3090 --label node.yovole.com/type=gpu --label node.yovole.com/role=worker --address 10.4.132.154 --internal-address 100.71.50.154


bash /root/NVIDIA-Linux-x86_64-470.94.run -a -q --ui=none


```
{
    "max-concurrent-downloads": 10,
    "max-concurrent-uploads": 10,
    "mtu": 1450,
    "insecure-registries": ["harbor.yovole.tech", "harbor-ai.yovole.io"],
    "hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"],
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    },
    "storage-driver": "overlay2",
    "storage-opts": [
        "overlay2.override_kernel_check=true",
        "overlay2.size=20G"
    ],
    "log-driver": "json-file",
    "log-opts": {
        "max-size": "100m",
        "max-file": "3"
    },
    "graph": "/docker/"
}

distribution=$(. /etc/os-release;echo $ID$VERSION_ID)       && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-contain
er-toolkit-keyring.gpg       && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list |             sed 's#deb https://#deb [signed-by=/usr/share/keyr
ings/nvidia-container-toolkit-keyring.gpg] https://#g' |             sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

apt install initramfs-tools
update-initramfs -u

curl -s --cacert /etc/kubernetes/ssl/kube-ca.pem --cert /etc/kubernetes/ssl/kube-apiserver-proxy-client.pem --key /etc/kubernetes/ssl/kube-apiserver-proxy-client-key.pem  -H "Authorization: Bearer kubeconfig-user-rl758jc6qm:n2j6bppgvzvswj95z4c6wfgrclzr899bh5jc4d4r9xsmx55km6b4ms" --header "Content-Type: application/json-patch+json" --request PATCH  https://127.0.0.1:6443/api/v1/nodes/cn-east-3c-compute-52-155/status --data '[{"op": "add", "path": "/status/capacity/nvme.com~1cache", "value": "6000"}]'


docker ps -a --format {{.ID}} | xargs -i -t docker rm {}

docker images --format {{.ID}} | xargs -i -t docker rmi {}

docker volume ls --format {{.Name}} | xargs -i -t docker volume rm {}

mv /etc/kubernetes /etc/kubernetes-bak-$(date +"%Y%m%d%H%M")
mv /var/lib/etcd /var/lib/etcd-bak-$(date +"%Y%m%d%H%M")
mv /var/lib/rancher /var/lib/rancher-bak-$(date +"%Y%m%d%H%M")
mv /opt/rke /opt/rke-bak-$(date +"%Y%m%d%H%M")

# 删除残留路径
rm -rf /etc/cni \
    /opt/cni \
    /run/secrets/kubernetes.io \
    /run/calico \
    /run/flannel \
    /var/lib/calico \
    /var/lib/cni \
    /var/lib/kubelet \
    /var/log/containers \
    /var/log/kube-audit \
    /var/log/pods \
    /var/run/calico \
    /usr/libexec/kubernete
```