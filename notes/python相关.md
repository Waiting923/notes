# Python
## 虚拟环境
1. 安装
```shell
# pip3 install virtualenv
```
2. 生成虚拟环境
```shell
# virtualenv osc
```
3. 使用虚拟环境
```shell
加载虚拟环境
#source osc/bin/active
退出虚拟环境
#deactivate
```
## 查询库依赖
  1. 使用pipdeptree查询依赖树
```shell
# pip install pipdeptree
# pipdeptree -p python-openstackclient
```