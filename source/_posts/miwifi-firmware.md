---
title: 解包小米路由器固件
toc: true
date: 2023-11-01 00:36:27
updated: 2023-11-07 16:48:00
categories: 技术
tags:
- [小米]
- [路由器]
---

看到[Ljcbaby](https://github.com/ljcbaby)在群里说，小米路由器开ssh的[那个洞](https://zhuanlan.zhihu.com/p/460949138)在新机器可能依然存在，遂一起研究一下
<!-- more -->
> 本文中的操作主要在Ubuntu23.04中完成

先从[小米官网](http://miwifi.com/miwifi_download.html)上下载固件，没什么好说的（还在用http，离谱）。  
这里为了对比，下了ax3000t和ac2100的固件，这两台机器都有人在用，并且后者是已知有这个洞，而前者固件修复了这个问题但据说依然有办法。  
不清楚新固件有没有修，ac2100就用了[2.0.722](http://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/r2100/miwifi_r2100_firmware_4b519_2.0.722.bin)的`miwifi_r2100_firmware_4b519_2.0.722.bin`
## 解包[^1]
安装工具
``` bash
pip3 install ubi_reader
```
安装后
``` bash
ubireader_extract_images miwifi_r2100_firmware_4b519_2.0.722.bin
```

```
UBI_File Warning: end_offset - start_offset length is not block aligned, could mean missing data.UBI_File Warning: end_offset - start_offset length is not block aligned, could mean missing data.
```
有警告，不过问题不大，得到`img-1537728761_vol-ubi_rootfs.ubifs`
``` bash
unsquashfs img-1537728761_vol-ubi_rootfs.ubifs
```

```
Parallel unsquashfs: Using 6 processors
2763 inodes (2453 blocks) to write


create_inode: could not create character device squashfs-root/dev/console, because you're not superuser!
[========================================================================================================================================/ ] 5215/5216  99%

created 2422 files
created 219 directories
created 339 symlinks
created 0 devices
created 0 fifos
created 0 sockets
created 1 hardlink
```
解压出了整个文件系统
``` bash
ls
```
```
bin  data  dev  etc  lib  mnt  overlay  proc  readonly  rom  root  sbin  sys  tmp  userdisk  usr  var  www
```
代码在`/usr/lib/lua/xiaoqiang`里（好像代码层面小米路由器的代号是xiaoqiang？）  
ax3000t的.bin文件可以解出`img-1508723001_vol-kernel.ubifs`和`img-1508723001_vol-rootfs.ubifs`两个文件，但kernel那个用`unsquashfs`解不开，应该只是系统内核，意义不大，rootfs那个里面就有目录结构了

## 反编译
这里用了[一个github项目](https://github.com/NyaMisty/unluac_miwifi)，简单翻了下代码在`/src/unluac/`，但是没看到maven或者gradle之类的配置文件  
后来在actions配置（`/.github/workflows/jarbuild.yml`）里看到了构建方法，就是非常原始的一个个编译再拼起来  
``` bash
mkdir build
javac -d build -sourcepath src  src/unluac/*.java
jar -cfm build/unluac.jar src/META-INF/MANIFEST.MF -C build  .
```
得到编译产物unluac.jar，再扫两眼代码看下用法，~~让ChatGPT~~写个脚本
``` bash
# 创建decompiled目录（如果不存在）
mkdir -p decompiled

# 使用find命令递归查找source目录下的所有.lua文件
find source -type f -name "*.lua" -print0 | while IFS= read -r -d $'\0' file; do
    # 构造输出文件路径（替换source为decompiled）
    output_file="decompiled/$(dirname "${file/source/decompiled}")/$(basename "$file")"
    
    # 创建输出文件的目录（如果不存在）
    mkdir -p "$(dirname "$output_file")"
    
    # 使用unluac.jar工具进行反编译，将输出写入到对应的文件中
    java -jar ./unluac.jar "$file" > "$output_file"
    
    # 打印处理的文件路径
    echo "Decompiled: $file -> $output_file"
done
```
这是Windows的，但不知道为什么（可能是他程序的问题）子目录下的识别不到
``` bat
@echo off
setlocal enabledelayedexpansion

REM 设置源目录和目标目录
set "sourceDir=source"
set "outputDir=decompiled"

REM 创建目标目录
mkdir %outputDir% 2>nul

REM 递归处理源目录下的所有.lua文件
for /r "%sourceDir%" %%f in (*.lua) do (
    set "sourceFile=%%f"
    set "outputFile=!sourceFile:%sourceDir%=%outputDir%!"
    java -jar unluac.jar "!sourceFile!" > "!outputFile!"
)

echo "反编译完成"
pause
```
至此，我们就获得了小米路由器的源码，虽然经过了混淆但还是有一定的意义，可以进行一些简单的字符串搜索或者扔给AI

**2023.11.7更新**
ljc看到称bug存在的那位说出了方法，就不再研究了  
但是因为群是比较封闭的小群，也没问，所以就不把方法写出来了

[^1]: [某论坛的一个帖子](https://bbs.hassbian.com/thread-17298-1-1.html)
