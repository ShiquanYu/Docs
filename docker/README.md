# 这可能是史上最全的 (ubuntu)docker 开发环境搭建教程

docker 功能太多，不胜其扰，写个文档记录下。

本文档主要针对于 nvidia-docker 的开发环境搭建，基础的 docker 使用详见其[官方文档](https://docs.docker.com/engine/)。

## 安装 docker/nvidia-docker

详细安装方法可参考 [nvidia-docker 官方文档](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/overview.html)（打不开搭个梯子），此处仅添加避坑指南。

### 安装过程无法添加 GPG key

搭梯子，然后进行添加。

## 普通用户获取 docker root 权限

1. 添加docker用户组(一般安装docker时会自动添加)

    ```shell
    sudo groupadd docker
    ```

1. 将指定用户添加到docker用户组中 注:将`${USER}`替换为自己的用户名

    ```shell
    sudo gpasswd -a ${USER} docker
    ```

1. 重启docker服务

    ```shell
    sudo systemctl restart docker
    ```

1. 如还没有权限请`log out`并重新登录

## 使用

### 基础使用方法

创建容器：

```shell
docker run --gpus all \
--net host \
--privileged \
--name=${container_name} \
-it ${docker_image} \
/bin/bash
```

### 挂载主机目录

挂载后主机和容器可共享目录，实现主机与容器文件交互。

相关参数介绍：

```
-v ${主机目录}:${docker 中的目录}
```

- **TIPS:** 挂载的`主机目录`最好与`docker 中的目录`相同，方便路径复制。
- **TIPS2:** （**该操作存在风险，容器内默认有 root 权限，万一误操作 `rm -r /*` 可能引擎灾难性后果，不推荐新手使用。**）可直接在 docker 内挂载主机根目录，然后进入容器使用软链接`ln -s`链接各个文件夹。

完整命令：

```shell
docker run --gpus all \
--net host \
--privileged \
-v /path1:/path1 \
-v /path2:/path2 \
--name=${container_name} \
-it ${docker_image} \
/bin/bash
```

### 挂载设备（摄像头、串口等）

挂载设备后可在容器内访问相关设备。

相关参数介绍：

```
--device ${设备路径}
```

- **TIP:** 该命令也可挂载根目录后使用软链接链接相关设备。

完整命令：

```shell
# 在容器内挂载摄像头
docker run --gpus all \
--net host \
--privileged \
-v /path1:/path1 \
-v /path2:/path2 \
--device /dev/video0 \
--name=${container_name} \
-it ${docker_image} \
/bin/bash
```
