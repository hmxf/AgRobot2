# Compile Install of ROS Noetic and other Dependencies on `MIIVII EVO ORIN`

本文档记录了 [AgRobot2](https://github.com/hmxf/AgRobot2) 项目的 ROS 编译安装过程和依赖安装流程。

目前的开发工作在多种平台上进行，但实际运行将基于 `MIIVII EVO ORIN 32G` 设备。

该设备当前最新版本的镜像为 `5.1.1-2.2.0.45` 版本，设备信息如下：

- uname -a
    - Linux EVO-Orin 5.10.104-tegra #1 SMP PREEMPT Thu Sep 21 15:06:01 CST 2023 aarch64 aarch64 aarch64 GNU/Linux
- cat /etc/miivii_release
    - MIIVII EVO ORIN 5.1.1-2.2.0.45

本文所列出的工作均为标准操作的扩展，也就是说，在完成标准操作的对应步骤以外，还需要额外执行本文对应章节所列出的操作，未提到的章节按照文档顺序依次执行即可。

**务必注意由于环境不同所导致的路径变化，执行脚本前务必确认其内容**

## 失效源修改

- 米文官方源报 `404` 错误，暂时禁用掉

    ```bash
    sudo mv /etc/apt/sources.list.d/miivii-l4t-apt-source.list /etc/apt/sources.list.d/miivii-l4t-apt-source.list.bak
    ```

## 编译安装 ROS Noetic

- 创建编译路径

    在本项目中，公共步骤中提到的 `$ROS_BUILD_PATH` 被指定为 `/ros/build` 路径，而 `$ROS_INSTALL_PATH` 被指定为 `/ros` 路径，且计划将其存储于独立的磁盘内，因此需要添加挂载信息。

    ```bash
    sudo mount /dev/nvme0n1 /ros && sudo rm -rf /ros/lost+found
    sudo chown -R $USER:$USER /ros
    ```

    挂载完成后即可按照公共步骤提供的命令创建编译路径。

    **务必记得为 `/dev/nvme0n1` 添加开机自动挂载，否则重启后 ROS 将无法正常工作**

    开机自动挂载功能可以通过将以下行写入 `/etc/fstab` 文件并重启完成，务必注意盘符和路径可能需要按照实际情况修改

    ```
    /dev/nvme0n1  /ros  ext4  defaults  0  1
    ```

    重启后使用 `mount -l | grep ros` 确认挂载情况，出现如下输出则表示开机自动挂载配置成功且已成功挂载

    ```
    /dev/nvme0n1 on /ros type ext4 (rw,relatime,stripe=32)
    ```

- 使用已保存的 ROS 源码目录

    ```bash
    git clone https://github.com/hmxf/ROSource $ROS_BUILD_PATH
    tail noetic-AgRobot.rosinstall
    ```

- 添加其他需要的包并解决依赖

    ***除非你知道自己在做什么，否则建议首次安装时跳过该步骤，以免因为依赖问题导致部分包同时被编译安装和包管理器安装（注意选择正确的 Tag 或分支）***

    [AgRobot2](https://github.com/hmxf/AgRobot2) 项目的依赖树和版本需求

    - [ros_ht_msg](https://github.com/hmxf/ros_ht_msg)
        - 无
    - [ros_modbus_msg](https://github.com/hmxf/ros_modbus_msg)
        - 无（虽然本项目依赖 `pymodbus` 包的特定版本，但该包并非 ROS 包）
    - [ros_bms_msg](https://github.com/hmxf/ros_bms_msg)
        - [ros-noetic-serial](https://github.com/wjwwood/serial/tree/1.2.1)
    - [imu_gps_fusion](https://github.com/hmxf/imu_gps_fusion)
        - [ros-noetic-catkin-virtualenv](https://github.com/locusrobotics/catkin_virtualenv/tree/0.6.1)
        - [ros-noetic-gps-common](https://github.com/swri-robotics/gps_umd/tree/0.3.4)
        - [ros-noetic-navigation 和 ros-noetic-move-base](https://github.com/ros-planning/navigation/tree/1.17.3)
            - [ros-noetic-navigation-msgs](https://github.com/ros-planning/navigation_msgs/tree/1.14.1)

- 检查依赖情况

    执行该步骤需要时刻注意终端提示内容，如果需要通过 apt 安装 `ros-noetic-*` 包，需要取消安装并记录这些包，再从 `解决依赖` 步骤开始重新分析依赖关系并获取源码，直到以下命令提示所有依赖都可以满足。

    ```bash
    rosdep install --from-paths ./src --ignore-packages-from-source --rosdistro noetic
    #All required rosdeps installed successfully
    ```

- 创建编译目标路径并编译安装 ROS

    在 Ubuntu 系统中，通过 APT 安装的 ROS 默认路径位于 `/opt/ros/noetic` 中，因此强烈不建议将源码编译的 ROS 环境同样定位到该路径，建议选择其他有写入权限的路径。

    ```bash
    mkdir -p /ros/noetic
    ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release -DPYTHON_EXECUTABLE=/usr/bin/python3 --install-space /ros/noetic
    ```

- 如果要更新已编译安装完成的 ROS 则需要从 `获取 ROS 源码` 步骤开始重新执行整个编译安装过程，如果要添加新的 ROS 包则需要从 `解决依赖` 步骤开始重新执行整个编译安装过程。


