# NVIDIA GPU HPL benchmark

## 安装步骤（参考链接[1]）

1. 确认系统内安装好新版本cuda驱动
```shell
$ nvidia-smi
Tue Dec  7 10:12:29 2021
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 495.29.05    Driver Version: 495.29.05    CUDA Version: 11.5     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================|
|   0  Tesla M60           Off  | 00000000:4A:00.0 Off |                  Off |
| N/A   48C    P0    41W / 150W |      0MiB /  8129MiB |      0%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
|   1  Tesla M60           Off  | 00000000:4B:00.0 Off |                  Off |
| N/A   34C    P0    38W / 150W |      0MiB /  8129MiB |      1%      Default |
|                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

2. 在安装docker前先安装nvidia-docker2以及nvidia-container-toolkit(参考链接[2])
关联包如下
```shell
$ rpm -qa | grep nvidia
nvidia-container-toolkit-1.7.0-1.x86_64
libnvidia-container-tools-1.7.0-1.x86_64
nvidia-docker2-2.8.0-1.noarch
libnvidia-container1-1.7.0-1.x86_64
```
安装后再安装docker，若docker先安装，则需要重启docker

3. 使用nvidia账户拉取hpc-benchmarks镜像(参考链接[3])
```shell
$ docker login nvcr.io

Username: $oauthtoken
Password: Your NGC API Key

docker pull nvcr.io/nvidia/hpc-benchmarks:21.4-hpl
```
---

## 使用步骤(参考链接[1][4])
1. 创建容器映射路径，生成映射变量
```shell
$ cat src
#!/bin/bash
CONT='nvcr.io/nvidia/hpc-benchmarks:21.4-hpl'
MOUNT="/root/wzh/hpl:/hpl"
```

2. 在映射路径中创建HPL.dat文件以及config文件
对应文件参数自行配置
```shell
$ cat HPL.dat
HPLinpack benchmark input file
Innovative Computing Laboratory, University of Tennessee
HPL.out      output file name (if any)
8            device out (6=stdout,7=stderr,file)
1            # of problems sizes (N)
10000          Ns
1            # of NBs
128 192 232 256 336 384      NBs
1            PMAP process mapping (0=Row-,1=Column-major)
1            # of process grids (P x Q)
1 2 4       Ps
2 24 12         Qs
16.0         threshold
1            # of panel fact
2 1 0        PFACTs (0=left, 1=Crout, 2=Right)
1            # of recursive stopping criterium
2            NBMINs (>= 1)
1            # of panels in recursion
2            NDIVs
1            # of recursive panel fact.
1 0 2        RFACTs (0=left, 1=Crout, 2=Right)
1            # of broadcast
0            BCASTs (0=1rg,1=1rM,2=2rg,3=2rM,4=Lng,5=LnM)
1            # of lookahead depth
0            DEPTHs (>=0)
0            SWAP (0=bin-exch,1=long,2=mix)
1            swapping threshold
1            L1 in (0=transposed,1=no-transposed) form
0            U  in (0=transposed,1=no-transposed) form
0            Equilibration (0=no,1=yes)
8            memory alignment in double (> 0)
```

```shell
$ cat M60
#!/bin/bash
CPU_AFFINITY="44-47:0-3"
CPU_CORES_PER_RANK=4
GPU_AFFINITY="0:1"
MEM_AFFINITY="1:0"
NET_AFFINITY="mlx5_bond_0"
GPU_CLOCK="1380,1410"
```

3. 加载变量运行容器
```shell
$ source src
$ docker run --name nvidia-hpl --privileged --gpus all -v ${MOUNT} ${CONT} mpirun --bind-to none -np 2 hpl.sh --config /hpl/M60 --dat /hpl/HPL.dat
```


## _参考链接_：

[1]_https://catalog.ngc.nvidia.com/orgs/nvidia/containers/hpc-benchmarks_

[2]_https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html#pre-requisites_

[3]_https://docs.nvidia.com/ngc/ngc-private-registry-user-guide/index.html#generating-api-key_

[4]_http://hpl-calculator.sourceforge.net/Howto-HPL-GPU.pdf_