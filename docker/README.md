# 这可能是史上最全的 (ubuntu)docker 开发环境搭建教程

docker 功能太多，不胜其扰，写个文档记录下。

本文档主要针对于 nvidia-docker 的开发环境搭建，基础的 docker 使用详见其[官方文档](https://docs.docker.com/engine/)。

## 目录

[安装 docker/nvidia-docker](#安装-dockernvidia-docker)
- [安装过程无法添加 GPG key](#安装过程无法添加-gpg-key)

[普通用户获取 docker root 权限](#普通用户获取-docker-root-权限)

[通过镜像创建容器](#通过镜像创建容器)
- [基础使用方法](#基础使用方法)
- [挂载宿主机目录](#挂载宿主机目录)
- [挂载设备（摄像头、串口等）](#挂载设备摄像头串口等)
- [容器开启 X11 服务，在宿主机显示画面](#容器开启-x11-服务在宿主机显示画面)
- [容器开启 IPC 通信](#容器开启-ipc-通信)

[启动容器](#启动容器)

[保存/导入镜像/容器](#保存导入镜像容器)
- [保存容器/镜像](#保存容器镜像)
- [导入镜像](#导入镜像)

[常见问题](#常见问题)

## 安装 docker/nvidia-docker

详细安装方法可参考 [nvidia-docker 官方文档](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/overview.html)（打不开搭个梯子），此处仅添加避坑指南。

### 安装过程无法添加 GPG key

- 搭梯子，然后进行添加。

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

## 通过镜像创建容器

镜像一般直接去[docker hub](https://hub.docker.com/)搜索下拉即可。

### 基础使用方法

- 创建容器：

    ```shell
    docker run --gpus all \
    --net host \
    --privileged \
    --name=${container_name} \
    -it ${docker_image} \
    /bin/bash
    ```

### 挂载宿主机目录

- 挂载后宿主机和容器可共享目录，实现宿主机与容器文件交互。
- 相关参数介绍：

    ```
    -v ${宿主机目录}:${docker 中的目录}
    ```

- **TIPS:** 挂载的`宿主机目录`最好与`docker 中的目录`相同，方便路径复制。
- **TIPS2:** （**该操作存在风险，容器内默认有 root 权限，万一误操作 `rm -r /*` 可能引擎灾难性后果，不推荐新手使用。**）可直接在 docker 内挂载宿主机根目录，然后进入容器使用软链接`ln -s`链接各个文件夹。

- 完整命令：

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

- 挂载设备后可在容器内访问相关设备。
- 相关参数介绍：

    ```
    --device ${设备路径}
    ```

- **TIP:** 该命令也可挂载根目录后使用软链接链接相关设备。

- 完整命令：

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

### 容器开启 X11 服务，在宿主机显示画面

- 开启 X11 服务后就可以显示画面，如摄像头画面或显示图像(暂且还不知道如何将远程服务器上的 docker 画面在本地显示)。
- 相关参数介绍：

    ```
    -e DISPLAY=$DISPLAY
    -v /tmp/.X11-unix/:/tmp/.X11-unix/
    ```

- 完整命令：

    ```shell
    docker run --gpus all \
    --net host \
    --privileged \
    -v /path1:/path1 \
    -v /path2:/path2 \
    --device /dev/video0 \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix/:/tmp/.X11-unix/ \
    --name=${container_name} \
    -it ${docker_image} \
    /bin/bash
    ```

- **NOTE:** 开启XServer(在docker中开启图形化服务)，必须先在宿主机中运行:`xhost +`
- **NOTE2:** 若出现错误`X Error: BadShmSeg (invalid shared segment parameter)`或图像显示微灰色，在容器的环境变量中添加:

    ```
    export QT_X11_NO_MITSHM=1
    ```

### 容器开启 IPC 通信

- 某些应用（像分布式计算这种）需要连接外部网络，需要 docker 连接宿主机网络。(暂时还不知道为啥非要加这句，总之加了 horovod 就跑起来了)
- 相关参数介绍：

    ```
    --ipc=host
    ```

- 完整命令：

    ```shell
    docker run --gpus all \
    --net host \
    --ipc=host \
    --privileged \
    -v /path1:/path1 \
    -v /path2:/path2 \
    --device /dev/video0 \
    -e DISPLAY=$DISPLAY \
    -v /tmp/.X11-unix/:/tmp/.X11-unix/ \
    --name=${container_name} \
    -it ${docker_image} \
    /bin/bash
    ```

## 启动容器

启动容器主要有两种方法，一种是使用`attach`命令，还有一种是使用`exec`命令，
区别是使用`attach`启动容器退出后容器会自动关闭，`exec`退出后容器仍会运行，
所以我更倾向于使用后者。

当然，使用`VScode`的 docker 插件就更简单直观多了，VScode yyds！

### `attach`

```shell
docker start ${CONTAINER}
docker attach ${CONTAINER}
```

### `exec`

```shell
docker start ${CONTAINER}
docker exec -it ${CONTAINER} bash
```

## 保存/导入镜像/容器

此处建议使用 `docker save/load` 命令来保存容器或镜像，使用 `docker export` 命令会出现无法找到显卡驱动的问题，
不推荐，所以这里不再介绍。

### 保存容器/镜像

- 将容器保存为镜像

    ```shell
    $ docker commit ${CONTAINER} ${IMAGE}
    ```

- 保存镜像(以下幸福二选一)

    ```shell
    $ docker save -o ${SAVE_FILE}.tar ${IMAGE}
    $ docker save ${IMAGE} > ${SAVE_FILE}.tar
    ```

### 导入镜像

```shell
$ docker load ${SAVE_FILE}.tar
```


## 常见问题

### 1. 安装完 opencv-python 后运行`cv2.imshow()`显示缺少相关库

```
apt-get install libglib2.0-dev
apt-get install libsm6
apt-get install libxrender1
apt-get install libxext-dev
```

### 2. 容器内默认用户都是具有 root 权限的，如何管理用户权限

众所周知，在让普通用户获取了 docker root 权限后，使用 docker 就不用每次都加`sudo`了，
但是这也引发连另一个问题，进入容器后默认用户为 root ，创建的文件及文件夹对宿主机都是只读状态，
某些软件(如`mpi/horovod`等)还会提示你不能使用 root 权限执行，同时安全性也有锁降低，那么如何
在容器内使用普通用户登录且解决以上麻烦呢。

1. 先给容器内 root 账户设置密码，以方便普通用户临时获取 root 权限（如已设置跳过此步骤）

    ```shell
    passwd ${ROOT}
    ```

1. 在**容器**内创建用户（用户名最好与宿主机用户名相同）

    ```shell
    $ adduser ${USER}
    # 接下来根据提示添加密码啥的
    ```

1. 查看**宿主机**用户的`gid`和`uid`，一般为四位数

    ```shell
    $ id ${USER}
    ```

1. 在**容器**内修改用户的`gid`和`uid`

    ```shell script
    # 修改uid
    $ usermod -u ${UID} ${USER}
    # 修改gid
    $ groupmod -g ${GID} ${USER}
    ```

1. 进入普通用户

    ```shell
    # 若已进入容器，在容器内执行
    su - ${USER}
    # 若在宿主机，则在进入容器时添加 --user 参数
    docker exet -it --user ${USER} ${CONTAINER} bash
    ```

此时在容器内创建文件或文件夹，在宿主机中看到即为可读可写状态了，与容器内状态一致。
