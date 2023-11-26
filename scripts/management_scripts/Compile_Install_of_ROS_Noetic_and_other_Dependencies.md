# Compile Install of ROS Noetic and other Dependencies

本文档记录了 [AgRobot2](https://github.com/hmxf/AgRobot2) 项目的 ROS 编译安装过程和依赖安装流程。

目前的开发工作在多种平台上进行，但实际运行将基于 `MIIVII EVO ORIN 32G` 设备。

该设备当前最新版本的镜像为 `5.1.1-2.2.0.45` 版本，设备信息如下：

- uname -a
    - Linux EVO-Orin 5.10.104-tegra #1 SMP PREEMPT Thu Sep 21 15:06:01 CST 2023 aarch64 aarch64 aarch64 GNU/Linux
- cat /etc/miivii_release
    - MIIVII EVO ORIN 5.1.1-2.2.0.45

## 配置系统源

- 米文官方源报 `404` 错误，暂时禁用掉

    ```bash
    sudo mv /etc/apt/sources.list.d/miivii-l4t-apt-source.list /etc/apt/sources.list.d/miivii-l4t-apt-source.list.bak
    ```

- 将默认 `ubuntu-ports` 源替换为清华源

    ```bash
    sudo mv /etc/apt/sources.list /etc/apt/sources.list.bak
    sudo vi /etc/apt/sources.list
    ```

    将以下信息写入 `/etc/apt/sources.list` 后保存退出

    ```
    # 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse

    deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
    # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse

    # 预发布软件源，不建议启用
    # deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
    # # deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
    ```

- 更新系统、安装常用工具

    ```bash
    sudo apt update && sudo apt update && sudo apt upgrade && sudo apt autoremove
    sudo apt install nano git zsh curl wget tree htop tmux screen can-utils net-tools openssh-server python3-pip
    ```

## 编译安装 ROS Noetic

- 设置软件源

    ```bash
    sudo sh -c 'echo "deb https://mirrors.tuna.tsinghua.edu.cn/ros/ubuntu $(lsb_release -sc) main" > /etc/apt/sources.list.d/ros-latest.list'
    curl -s https://raw.githubusercontent.com/ros/rosdistro/master/ros.asc | sudo apt-key add -
    sudo apt update
    ```

- 准备安装环境

    ```bash
    wget https://mirrors.tuna.tsinghua.edu.cn/ubuntu/pool/main/p/python-docutils/python3-docutils_0.20.1%2Bdfsg-2_all.deb
    wget https://mirrors.tuna.tsinghua.edu.cn/ubuntu/pool/main/p/python-docutils/docutils-common_0.20.1%2Bdfsg-2_all.deb
    wget https://mirrors.tuna.tsinghua.edu.cn/ubuntu/pool/main/p/python-docutils/docutils-doc_0.20.1%2Bdfsg-2_all.deb
    sudo dpkg -i docutils-common_0.20.1+dfsg-2_all.deb docutils-doc_0.20.1+dfsg-2_all.deb python3-docutils_0.20.1+dfsg-2_all.deb
    sudo apt install python3-rosdep python3-rosinstall-generator python3-vcstools python3-vcstool build-essential python3-catkin-tools libgoogle-glog-dev hugin-tools enblend glibc-doc
    sudo rosdep init && rosdep update
    ```

- 创建编译路径

    ```bash
    sudo mkdir /ros
    sudo mount /dev/nvme0n1 /ros && sudo rm -rf /ros/lost+found
    sudo chown -R $USER:$USER /ros && sudo chmod -R 755 /ros
    mkdir -p /ros/build/noetic && cd /ros/build/noetic
    ```
    **务必记得为 `/dev/nvme0n1` 添加开机自动挂载，否则 ROS 将无法正常工作**

    开机自动挂载功能可以通过将以下行写入 `/etc/fstab` 文件并重启完成，务必注意盘符和路径可能需要按照实际情况修改

    ```
    /dev/nvme0n1  /ros  ext4  defaults  0  1
    ```

    重启后使用 `mount -l | grep ros` 确认挂载情况，出现如下输出则表示开机自动挂载配置成功且已成功挂载

    ```
    /dev/nvme0n1 on /ros type ext4 (rw,relatime,stripe=32)
    ```



- 获取 ROS 源码目录

    ```bash
    rosinstall_generator desktop_full --rosdistro noetic --deps --tar > noetic-desktop_full.rosinstall
    ```

- 解决依赖

    以 [imu_gps_fusion](https://github.com/hmxf/imu_gps_fusion) 仓库为例，根据仓库中的描述，需要预先安装 `ros-noetic-catkin-virtualenv`、`ros-noetic-gps-common`、`ros-noetic-navigation`和`ros-noetic-move-base`四个包。解决依赖的具体步骤如下：

    - 查询各包的依赖，有以下三种方法，酌情选用

        1. 递归查询每个软件包的 `package.xml` 文件并记录其所需要的依赖，去重后分别查询其是否存在于 `noetic-desktop_full.rosinstall` 文件中。但人工查询在依赖树较为庞大时效率极低，推荐采用具备此类功能的脚本或程序自动化完成这一步骤。

        2. 当依赖树较为简单、依赖数量不多时，可以采用手动方式完成查询工作。

        3. 通过比对全部 ROS 包源码索引、本地已有的 ROS 包源码索引和所缺乏的包名列表，手动修改 `noetic-desktop_full.rosinstall` 文件并加入需要但没有的包信息，进而自动地完成 ROS 新包的添加和环境的更新。

        本例将以第二种方法为主、结合使用前两种方法作为示例来演示添加新源码包的流程，**实际上该演示并没有完全解决依赖问题，仅作为流程说明**。作为对前两种方法的结合和延伸，在理解了 `rosinstall_generator` 和 `vcs` 工具在整个源码获取流程中的作用和协作方式后，第三种方法是效率最高的方案，本例也使用该方法完全满足了依赖关系，因此推荐在熟悉前两种方法后尽早学会应用第三种方法解决大型项目中的依赖问题。

    - 查询包管理器中各包的版本

        ```bash
        apt search ros-noetic-catkin-virtualenv
            ros-noetic-catkin-virtualenv/focal 0.6.1-2focal.20210726.195032 arm64
        apt search ros-noetic-gps-common
            ros-noetic-gps-common/focal 0.3.4-1focal.20230620.193541 arm64
        apt search ros-noetic-navigation
            ros-noetic-navigation/focal 1.17.3-1focal.20231013.225901 arm64
        apt search ros-noetic-move-base
            ros-noetic-move-base/focal 1.17.3-1focal.20231013.201001 arm64
        ```

    - 在 ROS WiKi 中查询各包的源代码地址

        ```
        ros-noetic-catkin-virtualenv    https://github.com/locusrobotics/catkin_virtualenv
        ros-noetic-gps-common           https://github.com/swri-robotics/gps_umd
        ros-noetic-navigation           https://github.com/ros-planning/navigation
        ros-noetic-move-base            https://github.com/ros-planning/navigation
        ```

    - 阅读以上各代码仓库中特定版本代码中的 `package.xml` 文件，其中前两个包的所有依赖都已满足，后两个包则在源于同一个仓库，且还有一个对 `move_base_msgs` 的依赖没有满足，对其的查询使我们找到了 `ros-noetic-navigation-msgs` 这个包

        ```
        ros-noetic-navigation-msgs      https://github.com/ros-planning/navigation_msgs
        ```

        该仓库中包含了 `map_msgs` 和 `move_base_msgs` 两个包，而在 `noetic-desktop_full.rosinstall` 文件中则只有前者存在，因此我们需要将后者也添加到源码中

    - 此时项目依赖树如下

        - [imu_gps_fusion](https://github.com/hmxf/imu_gps_fusion)
            - [ros-noetic-catkin-virtualenv](https://github.com/locusrobotics/catkin_virtualenv/tree/0.6.1)
            - [ros-noetic-gps-common](https://github.com/swri-robotics/gps_umd/tree/0.3.4)
            - [ros-noetic-navigation 和 ros-noetic-move-base](https://github.com/ros-planning/navigation/tree/1.17.3)
                - [ros-noetic-navigation-msgs](https://github.com/ros-planning/navigation_msgs/tree/1.14.1)
                    - map_msgs
                    - move_base_msgs

    - 除 `ros-noetic-navigation-msgs` 之外的其余依赖较易获取，它们的共同点是 `noetic-desktop_full.rosinstall` 文件并未对这些包作出声明。
    
        包 `ros-noetic-navigation-msgs` 的处理方式较为特殊。在 `获取 ROS 源码` 步骤中生成的 `noetic-desktop_full.rosinstall` 文件中，虽然指明了需要安装版本为 `1.14.1` 的 `map_msgs` 包，在 `src/navigation_msgs` 路径下也有 `map_msgs` 文件夹，但 `noetic-desktop_full.rosinstall` 文件没有提及 `ros-noetic-move-base-msgs` 包，`src/navigation_msgs` 路径下也没有 `move_base_msgs` 文件夹。虽然可以通过删除 `src/navigation_msgs` 文件夹再重新获取全部源码的方式来获得 `ros-noetic-move-base-msgs` 包，但这种方式每次更新都需要手动获取源码，而且对源码的改动未能同步给 `noetic-desktop_full.rosinstall` 文件也使得该方法可能容易引发其他问题。

        若执意通过修改源码的方式安装完整的 ```navigation_msgs``` 包但不向 `noetic-desktop_full.rosinstall` 文件添加相关信息，可以使用以下命令：

        ```bash
        rm -rf src/navigation_msgs && cd src && git clone https://github.com/ros-planning/navigation_msgs && cd navigation_msgs && git checkout 1.14.1 && cd ../..
        ```
    
        **注意：每次在执行 `获取 ROS 源码` 后 `noetic-desktop_full.rosinstall` 文件的 `navigation_msgs` 相关 Tag 都有可能被更新，包括但不限于对子模块的版本、路径甚至是数目，因此需要在还未执行 `解决依赖` 步骤前修改相关内容，修改后务必确保所有依赖关系都已被满足方可继续执行编译安装步骤。**

        若希望使用一些更加优雅的方式，可以考虑使用以下方案。该方案希望通过对 `noetic-desktop_full.rosinstall` 文件做修改从而尽可能让编译出来的 ROS 环境符合预期且易于更新。因此，更新 `noetic-desktop_full.rosinstall` 文件的命令需要完成的功能有：

        - 在 `map_msgs` 包后插入 `move_base_msgs` 包的信息
        - 保证 `move_base_msgs` 包的版本与 `map_msgs` 包一致

        因此，可以在 `noetic-desktop_full.rosinstall` 文件中搜索 `map_msgs` 并将其所在段落的内容整体复制并插入到该段落后，再修改其中的包名即可，例如：

        ```bash
        nano noetic-desktop_full.rosinstall
        ```

        - 当前版本 `map_msgs` 的包信息如下：

            ```
            - tar:
                local-name: navigation_msgs/map_msgs
                uri: https://github.com/ros-gbp/navigation_msgs-release/archive/release/noetic/map_msgs/1.14.1-1.tar.gz
                version: navigation_msgs-release-release-noetic-map_msgs-1.14.1-1
            ```

        - 将下列信息加入 `noetic-desktop_full.rosinstall` 文件：

            ```
            - tar:
                local-name: navigation_msgs/move_base_msgs
                uri: https://github.com/ros-gbp/navigation_msgs-release/archive/release/noetic/move_base_msgs/1.14.1-1.tar.gz
                version: navigation_msgs-release-release-noetic-move_base_msgs-1.14.1-1
            ```

        - 如果是方法三，则在此处继续向 `noetic-desktop_full.rosinstall` 文件添加其他包信息，例如 `ros-noetic-serial` 和 `catkin_virtualenv` 等包。

        - 将 `noetic-desktop_full.rosinstall` 文件另存为 `noetic-AgRobot.rosinstall` 并退出。

        需要注意的是，此方法仍然需要手动修改 `.rosinstall` 文件的内容，但相比前一种方法可靠性已大幅提高，修改过程也极为简单。而且 `.rosinstall` 文件可以被保存和分发，从而便于利用已经保存好的文件完整复刻出整个 ROS 环境

    **在修改 `noetic-desktop_full.rosinstall` 文件之后，务必通过以下命令获取源码包**，建议首先执行此类修改，然后获取源码，之后再执行 `git clone` 类修改。

    ```bash
    mkdir src
    vcs import --input noetic-desktop_full.rosinstall ./src
    ```

- 添加其他需要的包

    ***除非你知道自己在做什么，否则建议首次安装时跳过该步骤，以免因为依赖问题导致部分包同时被编译安装和包管理器安装（注意选择正确的 Tag 或分支）***

    [AgRobot2]() 项目的依赖树和版本需求

    - [ros_ht_msg](https://github.com/hmxf/ros_ht_msg)
        - 无
    - [ros_modbus_msg](https://github.com/hmxf/ros_modbus_msg)
        - 无
    - [ros_bms_msg](https://github.com/hmxf/ros_bms_msg)
        - [ros-noetic-serial](https://github.com/wjwwood/serial/tree/1.2.1)
    - [imu_gps_fusion](https://github.com/hmxf/imu_gps_fusion)
        - [ros-noetic-catkin-virtualenv](https://github.com/locusrobotics/catkin_virtualenv/tree/0.6.1)
        - [ros-noetic-gps-common](https://github.com/swri-robotics/gps_umd/tree/0.3.4)
        - [ros-noetic-navigation 和 ros-noetic-move-base](https://github.com/ros-planning/navigation/tree/1.17.3)
            - [ros-noetic-navigation-msgs](https://github.com/ros-planning/navigation_msgs/tree/1.14.1)
            
    - 通过 `git clone` 方式添加：

        bash
        src && git clone https://github.com/wjwwood/serial && cd serial && git checkout 1.2.1 && cd ../..
        src && git clone https://github.com/ros-drivers/rosserial && cd rosserial && git checkout 0.9.2 && cd ../..
        src && git clone https://github.com/ros-industrial/ros_canopen && cd ros_canopen && git checkout 0.8.5 && cd ../..
        src && git clone https://github.com/locusrobotics/catkin_virtualenv && cd catkin_virtualenv && git checkout 0.6.1 && cd ../..
        src && git clone https://github.com/swri-robotics/gps_umd && cd gps_umd && git checkout 0.3.4 && cd ../..
        src && git clone https://github.com/ros-planning/navigation && cd navigation && git checkout 1.17.3 && cd ../..

    - 通过修改 `.rosinstall` 文件方式添加：

      <details>
      <summary>展开复制</summary>
      <pre><code>- tar:
          local-name: catkin_virtualenv
          uri: https://github.com/locusrobotics/catkin_virtualenv-release/archive/release/noetic/catkin_virtualenv/0.6.1-2.tar.gz
          version: catkin_virtualenv-release-release-noetic-catkin_virtualenv-0.6.1-2
      - tar:
          local-name: geometry2/tf2_sensor_msgs
          uri: https://github.com/ros-gbp/geometry2-release/archive/release/noetic/tf2_sensor_msgs/0.7.7-1.tar.gz
          version: geometry2-release-release-noetic-tf2_sensor_msgs-0.7.7-1
      - tar:
          local-name: gps_umd/gps_common
          uri: https://github.com/swri-robotics-gbp/gps_umd-release/archive/release/noetic/gps_common/0.3.4-1.tar.gz
          version: gps_umd-release-release-noetic-gps_common-0.3.4-1
      - tar:
          local-name: gps_umd/gps_umd
          uri: https://github.com/swri-robotics-gbp/gps_umd-release/archive/release/noetic/gps_umd/0.3.4-1.tar.gz
          version: gps_umd-release-release-noetic-gps_umd-0.3.4-1
      - tar:
          local-name: gps_umd/gpsd_client
          uri: https://github.com/swri-robotics-gbp/gps_umd-release/archive/release/noetic/gpsd_client/0.3.4-1.tar.gz
          version: gps_umd-release-release-noetic-gpsd_client-0.3.4-1
      - tar:
          local-name: navigation/amcl
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/amcl/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-amcl-1.17.3-1
      - tar:
          local-name: navigation/base_local_planner
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/base_local_planner/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-base_local_planner-1.17.3-1
      - tar:
          local-name: navigation/carrot_planner
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/carrot_planner/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-carrot_planner-1.17.3-1
      - tar:
          local-name: navigation/clear_costmap_recovery
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/clear_costmap_recovery/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-clear_costmap_recovery-1.17.3-1
      - tar:
          local-name: navigation/costmap_2d
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/costmap_2d/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-costmap_2d-1.17.3-1
      - tar:
          local-name: navigation/dwa_local_planner
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/dwa_local_planner/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-dwa_local_planner-1.17.3-1
      - tar:
          local-name: navigation/fake_localization
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/fake_localization/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-fake_localization-1.17.3-1
      - tar:
          local-name: navigation/global_planner
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/global_planner/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-global_planner-1.17.3-1
      - tar:
          local-name: navigation/map_server
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/map_server/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-map_server-1.17.3-1
      - tar:
          local-name: navigation/move_base
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/move_base/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-move_base-1.17.3-1
      - tar:
          local-name: navigation/move_slow_and_clear
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/move_slow_and_clear/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-move_slow_and_clear-1.17.3-1
      - tar:
          local-name: navigation/nav_core
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/nav_core/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-nav_core-1.17.3-1
      - tar:
          local-name: navigation/navfn
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/navfn/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-navfn-1.17.3-1
      - tar:
          local-name: navigation/navigation
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/navigation/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-navigation-1.17.3-1
      - tar:
          local-name: navigation/rotate_recovery
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/rotate_recovery/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-rotate_recovery-1.17.3-1
      - tar:
          local-name: navigation/voxel_grid
          uri: https://github.com/ros-gbp/navigation-release/archive/release/noetic/voxel_grid/1.17.3-1.tar.gz
          version: navigation-release-release-noetic-voxel_grid-1.17.3-1
      - tar:
          local-name: navigation_msgs/move_base_msgs
          uri: https://github.com/ros-gbp/navigation_msgs-release/archive/release/noetic/move_base_msgs/1.14.1-1.tar.gz
          version: navigation_msgs-release-release-noetic-move_base_msgs-1.14.1-1
      - tar:
          local-name: ros_canopen/can_msgs
          uri: https://github.com/ros-industrial-release/ros_canopen-release/archive/release/noetic/can_msgs/0.8.5-1.tar.gz
          version: ros_canopen-release-release-noetic-can_msgs-0.8.5-1
      - tar:
          local-name: ros_canopen/canopen_402
          uri: https://github.com/ros-industrial-release/ros_canopen-release/archive/release/noetic/canopen_402/0.8.5-1.tar.gz
          version: ros_canopen-release-release-noetic-canopen_402-0.8.5-1
      - tar:
          local-name: ros_canopen/canopen_chain_node
          uri: https://github.com/ros-industrial-release/ros_canopen-release/archive/release/noetic/canopen_chain_node/0.8.5-1.tar.gz
          version: ros_canopen-release-release-noetic-canopen_chain_node-0.8.5-1
      - tar:
          local-name: ros_canopen/canopen_master
          uri: https://github.com/ros-industrial-release/ros_canopen-release/archive/release/noetic/canopen_master/0.8.5-1.tar.gz
          version: ros_canopen-release-release-noetic-canopen_master-0.8.5-1
      - tar:
          local-name: ros_canopen/canopen_motor_node
          uri: https://github.com/ros-industrial-release/ros_canopen-release/archive/release/noetic/canopen_motor_node/0.8.5-1.tar.gz
          version: ros_canopen-release-release-noetic-canopen_motor_node-0.8.5-1
      - tar:
          local-name: ros_canopen/ros_canopen
          uri: https://github.com/ros-industrial-release/ros_canopen-release/archive/release/noetic/ros_canopen/0.8.5-1.tar.gz
          version: ros_canopen-release-release-noetic-ros_canopen-0.8.5-1
      - tar:
          local-name: ros_canopen/socketcan_bridge
          uri: https://github.com/ros-industrial-release/ros_canopen-release/archive/release/noetic/socketcan_bridge/0.8.5-1.tar.gz
          version: ros_canopen-release-release-noetic-socketcan_bridge-0.8.5-1
      - tar:
          local-name: ros_canopen/socketcan_interface
          uri: https://github.com/ros-industrial-release/ros_canopen-release/archive/release/noetic/socketcan_interface/0.8.5-1.tar.gz
          version: ros_canopen-release-release-noetic-socketcan_interface-0.8.5-1
      - tar:
          local-name: rosserial/rosserial
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_arduino
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_arduino/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_arduino-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_chibios
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_chibios/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_chibios-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_client
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_client/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_client-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_embeddedlinux
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_embeddedlinux/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_embeddedlinux-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_mbed
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_mbed/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_mbed-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_msgs
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_msgs/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_msgs-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_python
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_python/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_python-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_server
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_server/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_server-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_tivac
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_tivac/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_tivac-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_vex_cortex
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_vex_cortex/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_vex_cortex-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_vex_v5
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_vex_v5/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_vex_v5-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_windows
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_windows/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_windows-0.9.2-1
      - tar:
          local-name: rosserial/rosserial_xbee
          uri: https://github.com/ros-gbp/rosserial-release/archive/release/noetic/rosserial_xbee/0.9.2-1.tar.gz
          version: rosserial-release-release-noetic-rosserial_xbee-0.9.2-1
      - tar:
          local-name: serial
          uri: https://github.com/wjwwood/serial-release/archive/release/noetic/serial/1.2.1-1.tar.gz
          version: serial-release-release-noetic-serial-1.2.1-1
      </code></pre>
      </details>

- 检查依赖情况

    执行该步骤需要时刻注意终端提示内容，如果该步骤在解决依赖的过程中需要通过 apt 安装 `ros-noetic-*` 包，需要取消安装并记录这些包，再从 `解决依赖` 步骤开始重新分析依赖关系并获取源码，直到以下命令提示所有依赖都可以满足。

    ```bash
    rosdep install --from-paths ./src --ignore-packages-from-source --rosdistro noetic
    #All required rosdeps installed successfully
    ```

- 编译安装 ROS

    ```bash
    mkdir -p /ros/noetic
    ./src/catkin/bin/catkin_make_isolated --install -DCMAKE_BUILD_TYPE=Release -DPYTHON_EXECUTABLE=/usr/bin/python3 --install-space /ros/noetic
    ```

- 如果要更新已编译安装完成的 ROS 则需要从 `获取 ROS 源码` 步骤开始重新执行整个编译安装过程，如果要添加新的 ROS 包则需要从 `解决依赖` 步骤开始重新执行整个编译安装过程

## 安装 pyModbus

- 获取 pyModbus 源码

    ```bash
    cd ~
    git clone git://github.com/pymodbus-dev/pymodbus
    cd pymodbus && git checkout v3.5.2
    ```

- 准备安装环境

    ```bash
    sudo apt install python3-pip
    pip install --upgrade docutils==0.18.1
    pip install -r requirements.txt
    ```

- 安装 pyModbus

    ```bash
    pip install -e .
    pre-commit install
    ```

- 确认安装结果

    ```bash
    pip show pymodbus
    ```