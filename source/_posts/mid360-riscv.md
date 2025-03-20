---
title: 在RISC-V上使用mid360
toc: true
date: 2025-01-29 02:58:39
categories: 技术
cover: https://s21.ax1x.com/2025/01/29/pEVNiJP.md.jpg
excerpt: 没想到啊，这么顺利？
---

# 起因

因为学习需要用了一段时间的[ROS2](https://www.ros.org/)，又因为实习工作原因在用RISC-V架构的开发板，于是自然而然的想到了在RISC-V上使用ROS2

刚巧[RevyOS完成了适配](https://docs.revyos.dev/desktop/software/ROS2/)，在手头的LicheePi4A上安装后发现一切都很顺利，就想尝试实际应用一下了

众所周知啊，ROS2是用于机器人相关开发的，必然要使用到很多硬件；但在此同时呢，绝大多数硬件开发时完全不可能考虑到它在rv架构的兼容性，所以恐怕都比较寄。但是呢，手头刚好有一些mid360激光雷达，它使用TCP/IP来传输数据，兼容性上不容易出问题；同时它也不像工业相机那样有很大的数据量，这块~~性能羸弱的~~开发板应该可以正常收发，于是简单尝试了一下

# 运行

非常出人意料的是，[Livox提供的sdk](https://github.com/Livox-SDK/Livox-SDK2)~~的屎代码~~虽然使用高版本gcc无法编译[^1]，但是我切换到RevyOS源中的gcc-12后就可以正常编译了，只有几个`WARNING`，与x86上无异

[^1]: 这是我之前就发现的问题，在x86_64也是这样的，与risc-v无关

# 结果

然后运行[^2]`livox-ros-driver2`，就可以在rviz中看到雷达获取的点云了

> 这里要吐槽一下，他家有`livox_ros_driver` `livox_ros_driver2` `livox_ros2_driver`，不光代码很屎山，名字还起的这么绕

[^2]: 这里使用的不是[官方仓库](https://github.com/Livox-SDK/livox_ros_driver2)的，而是[一个第三方修改版](https://github.com/SMBU-PolarBear-Robotics-Team/livox_ros_driver2)，实际功能等都没有变化，只是让代码变得没有那么💩。我后来还提交了[一个PR](https://github.com/SMBU-PolarBear-Robotics-Team/livox_ros_driver2)

![](https://s21.ax1x.com/2025/01/29/pEVNpdA.jpg)

用`ros2 topic hz`查看点云的帧率是正常的，但在rviz中看非常卡，如果要移动视角的话更是低于1fps

这大概是因为缺少gpu驱动，这个xfce桌面没有使用图形加速；而cpu本身性能就不行，还要负担这么重的图形计算任务，结果就是非常糟糕的体验

等有了性能足够的芯片以及在图形方面足够的软件支持，或许可以尝试一下把现在做的东西迁移到risc-v上来

# 遇到的问题

这里也不是完全没有遇到问题，列出一些我遇到的供参考：

## CMake Warning

在CMake运行Findxxx.cmake找Eigen, PCL, Python, ROS等依赖时，都会报警告`Policy CMP0148 is not set`，参考[文档](https://cmake.org/cmake/help/latest/policy/CMP0148.html)手动改一下，或者直接设参数`-Wno-dev`忽略即可，不会影响最终结果

## rviz2报错AT-SPI

运行时可能会有AT-SPI报错

``` log
[rviz2-2] (rviz2:25390): dbind-WARNING **: 17:30:40.653: AT-SPI: Error retrieving accessibility bus address: org.freedesktop.DBus.Error.ServiceUnknown: The name org.a11y.Bus was not provided by any .service files
[ERROR] [rviz2-2]: process has died [pid 25390, exit code -11, cmd '/opt/ros/humble/lib/rviz2/rviz2 --display-config /home/debian/livox_ros_driver2/launch/../config/display_point_cloud.rviz --ros-args'].
```

使用apt安装`at-spi2-core`包即可解决

## rviz2报错Segmentation fault

启动rviz2时会报错`Segmentation fault`，无其他报错，即使将log-level设为DEBUG也看不到更多有用的信息

``` log
debian@revyos-lpi4a:~/livox_ros_driver2$ rviz2 --ros-args --log-level DEBUG
[DEBUG] [1742465374.358684709] [rclcpp]: signal handler installed
[DEBUG] [1742465374.358703710] [rclcpp]: deferred_signal_handler(): waiting for SIGINT/SIGTERM or uninstall
[DEBUG] [1742465374.359187059] [rcl]: Initializing node 'rviz' in namespace ''
[DEBUG] [1742465374.359362064] [rcl]: Using domain ID of '2'
[DEBUG] [1742465374.402647462] [rcl]: Initializing publisher for topic name '/rosout'
[DEBUG] [1742465374.402933138] [rcl]: Expanded and remapped topic name '/rosout'
[DEBUG] [1742465374.410872061] [rcl]: Publisher initialized
[DEBUG] [1742465374.411102068] [rcl]: Node initialized
[DEBUG] [1742465374.411547749] [rcl]: Initializing service for service name 'rviz/get_parameters'
[DEBUG] [1742465374.411706088] [rcl]: Expanded and remapped service name '/rviz/get_parameters'
[DEBUG] [1742465374.416014894] [rmw_fastrtps_cpp]: ************ Service Details *********
[DEBUG] [1742465374.416214900] [rmw_fastrtps_cpp]: Sub Topic rq/rviz/get_parametersRequest
[DEBUG] [1742465374.416312570] [rmw_fastrtps_cpp]: Pub Topic rr/rviz/get_parametersReply
[DEBUG] [1742465374.416398239] [rmw_fastrtps_cpp]: ***********
[DEBUG] [1742465374.416825587] [rcl]: Service initialized
[DEBUG] [1742465374.417099595] [rcl]: Initializing service for service name 'rviz/get_parameter_types'
[DEBUG] [1742465374.417228266] [rcl]: Expanded and remapped service name '/rviz/get_parameter_types'
[DEBUG] [1742465374.419828684] [rmw_fastrtps_cpp]: ************ Service Details *********
[DEBUG] [1742465374.420026023] [rmw_fastrtps_cpp]: Sub Topic rq/rviz/get_parameter_typesRequest
[DEBUG] [1742465374.420121360] [rmw_fastrtps_cpp]: Pub Topic rr/rviz/get_parameter_typesReply
[DEBUG] [1742465374.420198029] [rmw_fastrtps_cpp]: ***********
[DEBUG] [1742465374.420439370] [rcl]: Service initialized
[DEBUG] [1742465374.420626043] [rcl]: Initializing service for service name 'rviz/set_parameters'
[DEBUG] [1742465374.420747713] [rcl]: Expanded and remapped service name '/rviz/set_parameters'
[DEBUG] [1742465374.423464468] [rmw_fastrtps_cpp]: ************ Service Details *********
[DEBUG] [1742465374.423708142] [rmw_fastrtps_cpp]: Sub Topic rq/rviz/set_parametersRequest
[DEBUG] [1742465374.423797145] [rmw_fastrtps_cpp]: Pub Topic rr/rviz/set_parametersReply
[DEBUG] [1742465374.423875148] [rmw_fastrtps_cpp]: ***********
[DEBUG] [1742465374.424161157] [rcl]: Service initialized
[DEBUG] [1742465374.424362830] [rcl]: Initializing service for service name 'rviz/set_parameters_atomically'
[DEBUG] [1742465374.424478500] [rcl]: Expanded and remapped service name '/rviz/set_parameters_atomically'
[DEBUG] [1742465374.427344926] [rmw_fastrtps_cpp]: ************ Service Details *********
[DEBUG] [1742465374.427545599] [rmw_fastrtps_cpp]: Sub Topic rq/rviz/set_parameters_atomicallyRequest
[DEBUG] [1742465374.427635269] [rmw_fastrtps_cpp]: Pub Topic rr/rviz/set_parameters_atomicallyReply
[DEBUG] [1742465374.427713938] [rmw_fastrtps_cpp]: ***********
[DEBUG] [1742465374.427935945] [rcl]: Service initialized
[DEBUG] [1742465374.428139619] [rcl]: Initializing service for service name 'rviz/describe_parameters'
[DEBUG] [1742465374.428292290] [rcl]: Expanded and remapped service name '/rviz/describe_parameters'
[DEBUG] [1742465374.430942709] [rmw_fastrtps_cpp]: ************ Service Details *********
[DEBUG] [1742465374.431134382] [rmw_fastrtps_cpp]: Sub Topic rq/rviz/describe_parametersRequest
[DEBUG] [1742465374.431227718] [rmw_fastrtps_cpp]: Pub Topic rr/rviz/describe_parametersReply
[DEBUG] [1742465374.431306388] [rmw_fastrtps_cpp]: ***********
[DEBUG] [1742465374.431542062] [rcl]: Service initialized
[DEBUG] [1742465374.431731401] [rcl]: Initializing service for service name 'rviz/list_parameters'
[DEBUG] [1742465374.431875073] [rcl]: Expanded and remapped service name '/rviz/list_parameters'
[DEBUG] [1742465374.434697164] [rmw_fastrtps_cpp]: ************ Service Details *********
[DEBUG] [1742465374.434888170] [rmw_fastrtps_cpp]: Sub Topic rq/rviz/list_parametersRequest
[DEBUG] [1742465374.434986173] [rmw_fastrtps_cpp]: Pub Topic rr/rviz/list_parametersReply
[DEBUG] [1742465374.435067509] [rmw_fastrtps_cpp]: ***********
[DEBUG] [1742465374.435447855] [rcl]: Service initialized
[DEBUG] [1742465374.435672529] [rcl]: Initializing publisher for topic name '/parameter_events'
[DEBUG] [1742465374.435807200] [rcl]: Expanded and remapped topic name '/parameter_events'
[DEBUG] [1742465374.442996098] [rcl]: Publisher initialized
[DEBUG] [1742465374.443965796] [rcl]: Initializing subscription for topic name '/parameter_events'
[DEBUG] [1742465374.444178803] [rcl]: Expanded and remapped topic name '/parameter_events'
[DEBUG] [1742465374.445930193] [rcl]: Subscription initialized
[DEBUG] [1742465374.593761967] [rviz2]: Load pixmap at package://rviz_common/images/splash_overlay.png
Segmentation fault
```

解决方案在[RevyOS的文档中](https://docs.revyos.dev/docs/desktop/software/ROS2/#rviz2)有写，按文档说明操作即可

> 使用 `sudo switch-gl gl4es` 再重启后使用 `LIBGL_ALWAYS_SOFTWARE=true rviz2 here` 启动

为了在`ros2 launch`时启动rviz2，我用export命令设置环境变量
