# kubeproxy-noproxy
简单介绍一下背景，一套容器云的环境里面，用户的容器镜像仓库服务通过helm起在master节点，镜像服务通过istio由外部访问，用户的pod起在node节点，node节点需要访问用户的容器镜像仓库服务拉取镜像。需要通过Istio authorizationpolicy来做公网访问限制。

存在以下两个源地址保持的问题
1. 需要让ns级别的sidecar获取到源地址，这样ns的ap才能生效
2. 由istio-ingress进入的流量会由kube-proxy的ipvs rr规则进行转发，转发后源地址会变为转发节点，这样会导致ap按照istio-ingress的ep比例概率失效。
## Istio白名单配置
```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: policy
  namespace: xxxxxx
spec:
  action: ALLOW
  rules:
  - from:
    - source:
        remoteIpBlocks:
        - 100.84.0.0/15
        - 100.122.0.0/16
        - 100.123.0.0/16
        - 10.4.174.0/24
```
## 解决sidecar源地址保持问题
这个问题比较简单，只需要开启sidecar的tproxy即可。
```
#对应资源添加annotation
        sidecar.istio.io/interceptionMode: TPROXY
```
## 解决转发源地址保持问题
刚开始摸索了很多方案，多多少少都会存在些问题，这里就只提供最后的解决方式。
解决的思路就是只在istio节点上把svc转发到对应节点的后端，其他节点不生成对应的转发规则。

首先，只让istio节点把svc转发到本节点的后端，这个官方就有解决方式。
```
#修改istio-ingress svc
externalTrafficPolicy: Local
```
修改了以后对应节点的ipvs规则就只有对应节点的rs。但是其他节点还是会产生对应external ip的转发规则，由于其他节点没有ep，就会把包丢弃，这样就导致node节点无法访问服务。

简单看了一下kube-proxy对应ipvs的代码，是通过ipset和iptables来实现的externalTrafficPolicy: Local，我们只需要让这些节点的kubeproxy不纳管istio-ingress svc就可以达到我们的目标。

看kubeproxy的代码发现其实kubeproxy实现了忽略服务的功能，只不过没有写到文档里面。(以1.25版本代码为例)
```
kubernetes/cmd/kube-proxy/app/server.go
func (s *ProxyServer) Run() error {
    ...
    	noProxyName, err := labels.NewRequirement(apis.LabelServiceProxyName, selection.DoesNotExist, nil)
	if err != nil {
		return err
	}

	noHeadlessEndpoints, err := labels.NewRequirement(v1.IsHeadlessService, selection.DoesNotExist, nil)
	if err != nil {
		return err
	}

	labelSelector := labels.NewSelector()
	labelSelector = labelSelector.Add(*noProxyName, *noHeadlessEndpoints)

	// Make informers that filter out objects that want a non-default service proxy.
	informerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.LabelSelector = labelSelector.String()
		}))
    ...
}

kubernetes/pkg/proxy/apis/well_known_labels.go
const (
	// LabelServiceProxyName indicates that an alternative service
	// proxy will implement this Service.
	LabelServiceProxyName = "service.kubernetes.io/service-proxy-name"
)
```
如果我们直接添加这个label会发现，所有的节点都会忽略掉这个svc，所以我们实现的思路很简单，只需要在除了istio以外的节点上kubeproxy新增一个label，让这些节点来忽略istio-ingress svc
```
kubernetes/cmd/kube-proxy/app/server.go
func (s *ProxyServer) Run() error {
    ...
	noProxyName, err := labels.NewRequirement(apis.LabelServiceProxyName, selection.DoesNotExist, nil)
	if err != nil {
		return err
	}

	noHeadlessEndpoints, err := labels.NewRequirement(v1.IsHeadlessService, selection.DoesNotExist, nil)
	if err != nil {
		return err
	}

	istioNoProxyName, err := labels.NewRequirement(apis.IstioLabelServiceProxyName, selection.DoesNotExist, nil)
	if err != nil {
		return err
	}

	labelSelector := labels.NewSelector()
	labelSelector = labelSelector.Add(*noProxyName, *noHeadlessEndpoints, *istioNoProxyName)

	// Make informers that filter out objects that want a non-default service proxy.
	informerFactory := informers.NewSharedInformerFactoryWithOptions(s.Client, s.ConfigSyncPeriod,
		informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.LabelSelector = labelSelector.String()
		}))
    ...
}

kubernetes/pkg/proxy/apis/well_known_labels.go
const (
	// LabelServiceProxyName indicates that an alternative service
	// proxy will implement this Service.
	LabelServiceProxyName = "service.kubernetes.io/service-proxy-name"
    IstioLabelServiceProxyName = "service.kubernetes.io/istio-service-proxy-name"
)
```
然后我们根据不同label为除istio以外的节点跑新的kube-proxy代码。（编译流程参考[kubelet编译](https://github.com/Riverdd/notes/blob/master/notes/kubelet编译.md)）

运行成功后，在istio-ingress svc添加label 
```
service.kubernetes.io/istio-service-proxy-name: istio
```
验证你会发现除了istio节点按照externalTrafficPolicy: Local的方式代理istio-ingress svc，其他节点都不会代理istio-ingress svc。

大功告成!
