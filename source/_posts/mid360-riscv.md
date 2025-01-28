---
title: 在RISC-V上使用mid360
toc: true
date: 2025-01-29 02:58:39
categories: 技术
cover: https://s21.ax1x.com/2025/01/29/pEVNiJP.md.jpg
excerpt: 没想到啊，这么顺利？
---

因为学习需要用了一段时间的[ROS2](https://www.ros.org/)，又因为实习工作原因在用RISC-V架构的开发板，于是自然而然的想到了在RISC-V上使用ROS2

刚巧[RevyOS完成了适配](https://docs.revyos.dev/desktop/software/ROS2/)，在手头的LicheePi4A上安装后发现一切都很顺利，就想尝试实际应用一下了

众所周知啊，ROS2是用于机器人相关开发的，必然要使用到很多硬件；但在此同时呢，绝大多数硬件开发时完全不可能考虑到它在rv架构的兼容性，所以恐怕都比较寄。但是呢，手头刚好有一些mid360激光雷达，它使用TCP/IP来传输数据，兼容性上不容易出问题；同时它也不像工业相机那样有很大的数据量，这块~~性能羸弱的~~开发板应该可以正常收发，于是简单尝试了一下

非常出人意料的是，[Livox提供的sdk](https://github.com/Livox-SDK/Livox-SDK2)~~的屎代码~~虽然使用高版本gcc无法编译[^1]，但是我切换到RevyOS源中的gcc-12后就可以正常编译了，只有几个`WARNING`，与x86上无异

[^1]: 这是我之前就发现的问题，在x86_64也是这样的，与risc-v无关

然后运行[^2]`livox-ros-driver2`，就可以在rviz中看到雷达获取的点云了

> 这里要吐槽一下，他家有`livox_ros_driver` `livox_ros_driver2` `livox_ros2_driver`，不光代码很屎山，名字还起的这么绕

[^2]: 这里使用的不是[官方仓库](https://github.com/Livox-SDK/livox_ros_driver2)的，而是[一个第三方修改版](https://github.com/SMBU-PolarBear-Robotics-Team/livox_ros_driver2)，实际功能等都没有变化，只是让代码变得没有那么💩。我后来还提交了[一个PR](https://github.com/SMBU-PolarBear-Robotics-Team/livox_ros_driver2)

![](https://s21.ax1x.com/2025/01/29/pEVNpdA.jpg)

用`ros2 topic hz`查看点云的帧率是正常的，但在rviz中看非常卡，如果要移动视角的话更是低于1fps

这大概是因为缺少gpu驱动，这个xfce桌面没有使用图形加速；而cpu本身性能就不行，还要负担这么重的图形计算任务，结果就是非常糟糕的体验

等有了性能足够的芯片以及在图形方面足够的软件支持，或许可以尝试一下把现在做的东西迁移到risc-v上来
