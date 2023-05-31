# kubelet 编译
## 前置条件
- 对应版本golang环境
- 调试后的kubernetes源码
- 注意编译平台架构

## 编译
确认代码修改没有问题后切换到对应源码主路径
```
#方法1
$ cd /root/wzh/kubernetes/
$ make all WHAT=cmd/kubelet

#方法2
$ cd /root/wzh/kubernetes/cmd/kubelet
$ go build -v
```
编译成功后会在/root/wzh/kubernetes/cmd/kubelet下生成可执行二进制文件。

## 替换运行
```
$ ps -elf | grep kubelet
$ systemctl stop kubelet
$ cp ./kubelet /usr/bin/kubelet
$ systemctl start kubelet
$ kubectl get nodes
csk-demo-node-66       Ready    <none>          104d   v1.25.0-csk
```

## 指定编译版本号
若不指定编译版本号，则编译后运行的kubelet的版本号可能是一堆乱码

编译版本号处理规则参考/root/wzh/kubernetes/hack/lib/version.sh

```
$ cd /root/wzh/kubernetes/
#却认本地修改已经commit
$ git tag v1.25.0-csk c91ae4ece9500505473a1a9dc8d7a12a157b938a
$ source /root/wzh/kubernetes/hack/lib/version.sh
$ KUBE_GO_PACKAGE="k8s.io"
$ KUBE_ROOT=/root/wzh/kubernetes
$ KUBE_GO_PACKAGE=/root/wzh/kubernetes
#执行下面两个函数，会把gobuild的参数打印出来，并把version的信息记录在version中
$ kube::version::ldflags;kube::version::save_version_vars version
$ ldflags=$(kube::version::ldflags)
$ go build -v -ldflags="$ldflags"
```

多次执行的时候不知道为什么会出现多个dirty的后缀，估计是脚本问题，没有研究，可以使用version文件读入版本信息
```
$ KUBE_GIT_VERSION_FILE=/root/wzh/kubernetes/version

$ cat version 
KUBE_GIT_COMMIT='c91ae4ece9500505473a1a9dc8d7a12a157b938a'
KUBE_GIT_TREE_STATE='dirty'
KUBE_GIT_VERSION='v1.25.0-csk-dirty-dirty' #这里就是多出来的dirty使用的时候记得去掉
KUBE_GIT_MAJOR='1'
KUBE_GIT_MINOR='25+'

#再次执行的时候脚本会先从KUBE_GIT_VERSION_FILE文件中载入版本信息

$ kube::version::ldflags
```

# 参考
https://www.cnblogs.com/zhongpan/p/11966749.html