# H20 deepseek r1部署
目前部署的3套方案
1. ray+vllm
2. sglang
3. tensorrt-llm


## 环境信息
- H20(96G) 8 * 2
- IB(400G)业务 * 4 + IB(200G)存储 * 2
- Ubuntu22.04

## 系统优化
```
#关闭自动升级
$ sudo apt-mark hold linux-image-generic linux-headers-generic
$ sudo apt-mark showhold

#修改文件句柄数
$ vim /etc/security/limits.conf
root soft nofile 204800
root hard nofile 204800
* soft nofile 204800
* hard nofile 204800

#调整内核参数优化高并发
#单个套接字监听队列最大长度
$ echo "net.core.somaxconn = 65535" >> /etc/sysctl.conf
#tcp链接暂存syn请求最大数量
$ echo "net.ipv4.tcp_max_syn_backlog = 65535" >> /etc/sysctl.conf
$ sysctl -p
```
## Miniconda安装
```
$ mkdir ~/miniconda

$ cd ~/miniconda

$ wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

$ bash Miniconda3-latest-Linux-x86_64.sh
 
Please answer 'yes' or 'no':'
>>> yes

##设置安装路径
Miniconda3 will now be installed into this location:
/root/miniconda3
 
  - Press ENTER to confirm the location
  - Press CTRL-C to abort the installation
  - Or specify a different location below
 
[/root/miniconda3] >>>
 
 
##有提示"installation finished."即安装完成。另外询问是否进入终端就是（base）环境，如果不需要就no，但这里其实有个误区，这里是要有必要选择yes的，因为这是conda的初始化，如果不做这一步，conda指令是无法使用的，这里就先选yes，然后不想进入shell就是(base)环境，就再用conda config --set auto_activate_base false取消即可
 
You can undo this by running `conda init --reverse $SHELL`? [yes|no]
[no] >>> yes
no change     /usr/miniconda3/condabin/conda
no change     /usr/miniconda3/bin/conda
no change     /usr/miniconda3/bin/conda-env
no change     /usr/miniconda3/bin/activate
no change     /usr/miniconda3/bin/deactivate
no change     /usr/miniconda3/etc/profile.d/conda.sh
no change     /usr/miniconda3/etc/fish/conf.d/conda.fish
no change     /usr/miniconda3/shell/condabin/Conda.psm1
no change     /usr/miniconda3/shell/condabin/conda-hook.ps1
no change     /usr/miniconda3/lib/python3.12/site-packages/xontrib/conda.xsh
no change     /usr/miniconda3/etc/profile.d/conda.csh
modified      /root/.bashrc
 
==> For changes to take effect, close and re-open your current shell. <==
 
Thank you for installing Miniconda3!
 
## 生效
$ source /root/.bashrc
 
##配置 conda 国内镜像
##清华源
$ conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
$ conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
$ conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
$ conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
$ conda config --set show_channel_urls yes
 
##阿里源
$ conda config --add channels https://mirrors.aliyun.com/anaconda/pkgs/main/
$ conda config --add channels https://mirrors.aliyun.com/anaconda/pkgs/free/
$ conda config --set show_channel_urls yes
```
## NV驱动相关安装
```
#安装驱动与cuda
$ apt -y install gcc
$ apt -y install make
$ apt -y install dkms

$ wget https://developer.download.nvidia.com/compute/cuda/12.4.0/local_installers/cuda_12.4.0_550.54.14_linux.run
 
$ sudo sh cuda_12.4.0_550.54.14_linux.run

$ nvidia-smi --version

#安装nvitop
$ sudo apt -y install python3-pip
$ pip3 install nvitop
$ nvitop

#安装nvlink(nvidia-fabricmanager)
#nvidia-fabricmanager版本需要与cuda版本像对应
NV资源汇总网站：https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/

$ wget https://developer.download.nvidia.cn/compute/cuda/repos/ubuntu2004/x86_64/nvidia-fabricmanager-550_550.54.14-1_amd64.deb
$ apt-get install ./nvidia-fabricmanager-550_550.54.14-1_amd64.deb -y
$ systemctl enable nvidia-fabricmanager
$ systemctl restart nvidia-fabricmanager
$ systemctl status nvidia-fabricmanager
```

## 安装DOCKER
```
# Add Docker's official GPG key:
$ sudo apt update
$ sudo apt -y install ca-certificates curl
$ sudo install -m 0755 -d /etc/apt/keyrings
 
$ sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc   (如果失败了，多试几次..)
$ sudo chmod a+r /etc/apt/keyrings/docker.asc
 
 
# Add the repository to Apt sources:
$ echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
$ sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
$ sudo apt update

$ sudo apt -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 安装NVIDIA Container Toolkit
```
1. Set up the GPG key and configure apt to use NVIDIA Container Toolkit packages in the file /etc/apt/sources.list.d/nvidia-docker.list

$ distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
  
$ curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
 
$ curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list

2. Download information from all configured sources about the latest versions of the
packages and install the nvidia-container-toolkit package.

$ sudo apt update && sudo apt install -y nvidia-container-toolkit

3. Restart the Docker service.

$ sudo systemctl restart docker
```

## 模型部署
- 模型下载 [魔塔]_https://modelscope.cn/models/deepseek-ai/DeepSeek-R1_

### ray+vllm推理引擎
ray主要负责任务调度，分布式执行以及资源管理和硬件加速

- 参考
1. [分布式推理]_https://docs.vllm.ai/en/stable/serving/distributed_serving.html_
2. [vllm启动参数]_https://docs.vllm.ai/en/stable/serving/engine_args.html_
3. [自动化脚本]__

镜像
```
vllm/vllm-openai:v0.8.5
```
启动ray cluster
``` 
#head节点
docker run \
    --entrypoint /bin/bash \
    --network host \
    --name node \
    --ipc host --privileged --device=/dev/infiniband:/dev/infiniband \
    --shm-size 10.24g \
    --gpus all \
    -v "${模型本地所在路径}:/mnt" \
    -e GLOO_SOCKET_IFNAME=bond0 \
    -e NCCL_SOCKET_IFNAME=bond0 \
    -e NCCL_IB_HCA=mlx5_0,mlx5_1,mlx5_4,mlx5_5 \
    -e VLLM_HOST_IP="${本节点地址} \
    vllm/vllm-openai:v0.8.5 \
    -c ray start --block --head --port=6379

#worker节点
docker run \
    --entrypoint /bin/bash \
    --network host \
    --name node \
    --ipc host --privileged --device=/dev/infiniband:/dev/infiniband \
    --shm-size 10.24g \
    --gpus all \
    -v "${MODEL_PATH}:/mnt" \
    -e GLOO_SOCKET_IFNAME=bond0 \
    -e NCCL_SOCKET_IFNAME=bond0 \
    -e NCCL_IB_HCA=mlx5_0,mlx5_1,mlx5_4,mlx5_5 \
    -e VLLM_HOST_IP="${LOCAL_ADDRESS} \
    vllm/vllm-openai:v0.8.5 \
    -c ray start --block --address=${HEAD_NODE_ADDRESS}:6379

#检查ray集群状态
$ ray status
======== Autoscaler status: 2025-05-22 23:33:17.364741 ========
Node status
---------------------------------------------------------------
Active:
 1 node_36d0b8a47fa2816d2eb92f46a5b7247c6cbd5c65d78e9663b8c35a8a
 1 node_69b5e71cb5890aa830f75990f47c7070c4a40be43979e75081c32d65
Pending:
 (no pending nodes)
Recent failures:
 (no failures)

Resources
---------------------------------------------------------------
Usage:
 0.0/448.0 CPU
 16.0/16.0 GPU (16.0 used of 16.0 reserved in placement groups)
 0B/3.56TiB memory
 0B/372.53GiB object_store_memory

Demands:
 (no resource demands)
```
启动vllm
```
#这里启动以bf16为例，fp16/fp32需要对模型路径和参数进行调整
docker exec -d ${CONTAINER_NAME} bash -c "nohup vllm serve \
    mnt/share/deepseek-ai/DeepSeek-R1-bf16 \
    --served-model-name DeepSeek-R1 --dtype bfloat16 \
    --trust-remote-code --max-model-len 65536 \
    --enable_chunked_prefill \
    --quantization None --tensor-parallel-size 8 \ #张量并行与pipline并行乘积不能超过gpu总数
    --pipeline-parallel-size 2 --port 40000 \ #自定义对外服务端口
    --disable-log-requests --max-num-batched-tokens 12800 \
    --gpu-memory-utilization 0.9 > /var/log/vllm.log 2>&1 &"

#查看日志直至服务端口40000监听成功
```

### Sglang推理引擎
- 参考
1. [sglang安装]_https://docs.sglang.ai/start/install.html_
2. [sglang运行deepseek]_https://github.com/sgl-project/sglang/blob/main/benchmark/deepseek_v3/README.md_
3. [sglang参数]_https://github.com/sgl-project/sglang/blob/main/docs/backend/server_arguments.md_
4. [自动化脚本]__

镜像
```
lmsysorg/sglang:latest
```
启动sglang
```
#sglang需要加载RDMA模块提升性能，使用gpu间内存直接交换而不通过cpu内存来节约通信时间
$ sudo modprobe nvidia-peermem

docker run -d --gpus all \
    --name dsnode \
    --shm-size 512g \
    --network=host \
    -v ${MODEL_PATH}:/deepseek \
    --restart always \
    -p 40000:40000 \
    --privileged --device=/dev/infiniband:/dev/infiniband \
    -e NCCL_IB_HCA=mlx5_0,mlx5_1,mlx5_4,mlx5_5 \ #对应的IB口
    -e GLOO_SOCKET_IFNAME=bond0 \
    -e NCCL_SOCKET_IFNAME=bond0 \
    -e NCCL_DEBUG=INFO \
    --ipc=host \
    lmsysorg/sglang:v0.4.4-cu125 \
    python3 -m sglang.launch_server \
      --model-path /deepseek/DeepSeek-R1 \
      --served-model-name DeepSeek-R1 \
      --trust-remote-code \
      --tp 16 \
      --enable-metrics \
      --enable-cache-report \
      --show-time-cost \
      --mem-fraction-static 0.9 \
      --max-prefill-tokens 2000 \ #prefill数值调大会增加首token输出时间(TTFT time to first token)
      --max-total-tokens 65536 \
      --dist-init-addr ${主节点ip}}:20001 \ #主节点地址
      --nnodes 2 \ #总节点数量
      --node-rank 0 \ #主节点写0，后续节点依次增加
      --host 0.0.0.0 \
      --port 40000 \
      --enable-torch-compile --torch-compile-max-bs 8 \
      --stream-output \
      --allow-auto-truncate
```
### Tensorrt-llm推理引擎
- 参考
1. [自动化脚本]__

## 模型测试
```
#模型状态信息接口
$ curl http://127.0.0.1:40000/get_server_info | jq

#模型会话接口
curl -X 'POST' \
  'http://127.0.0.1:40000/v1/chat/completions' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "DeepSeek-R1",
    "messages": [
      {
        "role":"user",
        "content":"Hello! How are you?"
      },
      {
        "role":"assistant",
        "content":"Hi! I am quite well, how can I help you today?"
      },
      {
        "role":"user",
        "content":"思考一下长沙有什么好玩的?"
      }
    ],
    "top_p": 1,
    "n": 1,
    "max_tokens": 9500,
    "stream": true,
    "frequency_penalty": 1.0,
    "stop": ["hello"]
  }'
```

## OpenwebUI部署
- 参考
1. [官方文档]_https://docs.openwebui.com/getting-started/quick-start/_
