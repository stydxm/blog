---
title: AOSC OS上LoongArch ROS2移植
toc: true
date: 2025-10-01 11:52:18
categories: 技术
tags: ROS
cover:
---

想在机器人领域使用一下龙芯，但目前ROS2官方并未发预构建二进制包

调研时发现了[loongros2](https://loongros.cn/)，证明源码直接或小改后在龙芯上运行没有太大问题，遂决定开坑移植

<!--more-->

## 准备工作

### [colcon](http://colcon.readthedocs.io/)

colcon是一个用于构建软件包集的命令行工具，大家都用它来编译ROS2项目。因此虽然它不属于ROS的一部分，还是先把它打包出来

colcon是完全由Python写的，不涉及到编译，也没有架构问题。虽然包的数量不少（Ubuntu的`python3-colcon`加上推荐依赖是二十多个），但构建也不麻烦，直接手动写一下再复制复制粘贴几次

相关PR：https://github.com/AOSC-Dev/aosc-os-abbs/pull/12771

### [flake8](https://flake8.pycqa.org/)

flake8是一个代码检查工具，虽然没有直接用到，但ROS里很多Python包都依赖它，因此也要打包一下

相关PR：https://github.com/AOSC-Dev/aosc-os-abbs/pull/12817

### 其他包

主要涉及到mypy、lttng等一大堆东西

相关PR：https://github.com/AOSC-Dev/aosc-os-abbs/pull/12823

11.7更新：由于社区反馈建议分开，就拆成了多个PR

## ROS构建目标

在[所有的ROS版本中](https://docs.ros.org/en/rolling/Releases.html)，现在还在生命周期中的正式版有三个。其中Kilted不是LTS版本，而Humble比较老，依赖版本比较低，因此选择最新的LTS版本Jazzy。查看[文档中对依赖库版本的需求](https://www.ros.org/reps/rep-2000.html#jazzy-jalisco-may-2024-may-2029)均能满足，决定将目标distribution定为Jazzt Jalisco

[REP 2001](https://www.ros.org/reps/rep-2001.html#jazzy-jalisco-may-2024-may-2029)中规定了每个版本的不同变种，Jazzy有六个，其中最低的是ROS Core，其次是ROS Base

~~我暂时将目标Variant定为base，如进展顺利再尝试打包更多包以达到desktop~~

11.7更新：打包base时没有遇到太多问题，然后再花了一些时间打出了dekstop

## 编写脚本生成打包脚本

因为ROS的包数量大、相似度高，手工编写打包脚本远不如用脚本批量生成

个人比较熟悉Python，就用它来写了

### 编写软件包的类

参照[AOSC的指南](https://wiki.aosc.io/zh/developer/packaging/package-styling-manual/)，编写一个Python类

``` python
class Package:
    version: str
    src: str
    name: str
    dependencies: list[str]
    build_dependencies: list[str]
    description: str
    autobuild_type: str
```

### 获取包列表

ros官方提供了一个工具`rosinstall-generator`，可以生成包源码的结构化数据，我们可通过[提供的文档](https://wiki.ros.org/rosinstall_generator)来了解它的用法

使用pip安装后，尝试获取一份包含依赖项的core级别所有包的源码

``` bash
rosinstall_generator ros_core --rosdistro jazzy --deps
```

使用python解析

``` python
import subprocess
import sys
from typing import Literal

import yaml

raw_package_type = dict[
    Literal["git"], dict[Literal["local-name", "uri", "version"], str]
]
try:
    rosinstall_generator_output = subprocess.run(
        ["rosinstall_generator", "ros_base", "--rosdistro", "jazzy", "--deps"],
        capture_output=True,
        check=True,
    )
    raw_packages_str = rosinstall_generator_output.stdout
    raw_packages_data: raw_package_type = yaml.safe_load(raw_packages_str)
except subprocess.CalledProcessError as e:
    print(f"命令执行失败: {e.returncode}", file=sys.stderr)
    print(e.stderr)
except FileNotFoundError:
    print(f"找不到命令，rosinstall-generator 未安装", file=sys.stderr)
except yaml.YAMLError as e:
    print(f"返回值解析错误: {e}", file=sys.stderr)
```

### 解析包信息
#### 版本

读取到信息中的版本号均为`x.y.z-revision`的格式

按照[AOSC的规定](https://wiki.aosc.io/zh/developer/packaging/package-styling-manual/#ruan-jian-bao-ban-ben)：

> 情形：版本号带有短横线（-）
> 应当采取的措施：将短横线替换为加号（+）
> 举例：ImageMagick 6.9.10-23 -> VER=6.9.10+23

将其进行字符串替换，改为`x.y.z+revision`

#### 源码

`rosinstall_generator`输出的均为git仓库，因此只需要做简单拼接`"git::commit=tags/{version}::{uri}"`

但由于已经单独提出了`VER`变量，在`SRC`中再硬编码版本号就显得不太合适，所以稍作修改

``` python
semver = version.split("/")[-1]
self.version = semver.replace("-", "+")
version_prefix = "/".join(version.split("/")[:-1])
self.src = f"git::commit=tags/{version_prefix}/{'${VER/+/-}'}::{uri}"
```

这样处理后，例如ament_package这个包的生成内容就是

``` bash
VER=0.16.4+1
SRCS="git::commit=tags/release/jazzy/ament_package/${VER/+/-}::https://github.com/ros2-gbp/ament_package-release.git"
```

#### 更新检查

样式指南要求尽量使用anitya，但要把如此数量庞大的包都加到anitya过于麻烦，就选了[使用GitHub格式](https://github.com/AOSC-Dev/aosc-findupdate/blob/master/docs/config.zh-CN.md#github-api)

``` python
repo = uri[19:-4]
pattern = "/".join(version.split("/")[:-1]) + "/.*"
check_update = f"github::repo={repo};pattern={pattern}"
```

#### 包名

观察发现，rosinstall_generator输出的local-name都是`包所属“大包”名/包名`[^1]的格式，且包名一段是由下划线分割的。参考官方打包的deb包名，可**推测出**最终包名的格式

[^1]: 大包是我自己的不正规的叫法，指的是如ament等需要多个组件组成的包

``` python
original_name = local_name.split("/")[-1]
package_name = "ros-jazzy-" + original_name.replace("_", "-")
```

由于包名是推测的，没有找到相关文档，还需要进行验证

<details>
<summary>验证过程</summary>

利用ROS官方的apt仓库，通过检查目录是否存在来判断


``` python
import requests
from tqdm import tqdm

for package in tqdm(raw_packages_data):
    local_name = package["git"]["local-name"]
    original_name = local_name.split("/")[-1]
    package_name = "ros-jazzy-" + original_name.replace("_", "-")
    resp = requests.get(
        "http://packages.ros.org/ros2/ubuntu/pool/main/r/" + package_name
    )
    if resp.status_code != 200:
        print(local_name)
```
</details>

结果为全部通过，证实了我的猜测

#### 依赖

这是非常重要又很难处理的一部分，每个包都有一个`package.xml`文件[^2]，里面记录了包的依赖情况

[^2]: 该文件有三种版本，以`<package>`节点的`format`属性判断，分别是["1"](https://ros.org/reps/rep-0127.html)、["2"](https://ros.org/reps/rep-0140.html)、["3"](https://ros.org/reps/rep-0149.html)，观察发现好像版本2的比较多

依赖分为两种，编译时依赖和构建时依赖，用`<build_depend>`（编译依赖库）/`<buildtool_depend>`（编译时需要的编译工具）和`<depend>`（构建和运行都用）/`<exec_depend>`（只有运行用）表示

这些依赖有都有两个来源，其他的ros包和非ros的第三方库。ros包的命名方式很简单，通过前述生成包名的方式可以很容易找到；官方获取第三方库在包管理中的包名的实现方式是维护[一个大的映射表](https://github.com/ros/rosdistro/blob/master/rosdep/base.yaml)，我们使用其中的Arch作为基准，在此基础上手动进行修改：

- AOSC中的包通常不会以`-dev`或者`-devel`结尾，因此遇到该后缀就去掉

- AOSC打包的Python库不以`python3-`开头，因此遇到该前缀就去掉

- 上面两条中有个特例是Ubuntu中的包`python3-dev`，匹配上两条后啥都不剩了，所以要放在最前面做特殊处理

- AOSC打包的包名中若原来不含`lib`前缀的，不会加上该前缀，但本来就有的则会保留；若对比同一包在多个系统中的包名，有些带`lib`前缀而有些没有，就可以大胆猜测它不是原项目名的一部分，将它去除

~~这里用一个简单粗暴（但不一定准确）的方法来区分：含有下划线`_`的认为是ros包，带有横杠`-`的是第三方库。~~ 两种来源的包可以通过映射表中是否存在来判断

基于以上分析内容，写出python代码

``` python

import requests
import yaml

rosdep_map: dict[str, dict] = yaml.safe_load(
    requests.get(
        "https://github.com/ros/rosdistro/raw/refs/heads/master/rosdep/base.yaml"
    ).text
)


def process_rosdep_package_list(
    rosdep_package_list: list[str] | dict[str, list[str]],
) -> str:
    system_package_name = ""
    if isinstance(rosdep_package_list, list):
        system_package_name = rosdep_package_list[0]
    else:
        for system in rosdep_package_list.keys():
            if rosdep_package_list[system]:
                system_package_name = rosdep_package_list[system][0]
    assert system_package_name
    return system_package_name


def rosdep_key_to_package_name(rosdep_key: str) -> str:
    package_name = ""
    if rosdep_key in rosdep_map.keys():
        current_package = rosdep_map[rosdep_key]
        if "arch" in current_package:
            package_name = process_rosdep_package_list(current_package["arch"])
        elif "ubuntu" in current_package:
            package_name = process_rosdep_package_list(current_package["ubuntu"])
        else:
            package_name = process_rosdep_package_list(
                current_package[list(current_package.keys())[0]]
            )
    else:
        original_name = rosdep_key.split("/")[-1]
        package_name = f"ros-{rosdistro}-" + original_name.replace("_", "-")
    if package_name == "python3-dev" or package_name == "python3":
        package_name = "python-3"
    if package_name.startswith(f"ros-{rosdistro}-python3-"):
        package_name = package_name[13 + len(rosdistro) :]
    if package_name.startswith("python3-"):
        package_name = package_name[8:]
    if package_name.endswith("-dev"):
        package_name = package_name[:-4]
    return package_name
```

这里还要注意有个特殊的包`ros-jazzy-ros-workspace`，负责提供`/opt/ros/jazzy/setup.{sh,bash,zsh}`等脚本，它是除了本身及其依赖外所有包的依赖项

#### 描述

`package.xml`中有`<description>`字段包含了对包的描述；但它可能有多行，因此只取第一行

有个别包中的描述中含有反斜杠，需要注意转义

#### 构建方式

绝大多数包都是使用CMake编译或不需要编译的Python包，我选择根据是否存在`CMakeLists.txt`和`setup.py`判断；如遇到特殊情况手动处理

``` python
repo = uri[19:-4]
file_list = requests.get(f"https://api.github.com/repos/{repo}/contents?ref={version}").json()
has_pyproject = False
has_setup_py = False
has_cmake_lists = False
for file in file_list:
    if file["name"] == "pyproject.toml":
        has_pyproject = True
    if file["name"] == "setup.py":
        has_setup_py = True
    if file["name"] == "CMakeLists.txt":
        has_cmake_lists = True
if has_cmake_lists:
    self.build_type = "cmakeninja"
elif has_pyproject:
    self.build_type = "pep517"
elif has_setup_py:
    self.build_type = "python"
else:
    self.build_type = "unknown"
```

若包为Python写的，还要加上NOPYTHON2和noarch声明

### 生成ACBS文件

通过以上步骤，整合代码，写出完整程序

<details>
<summary>展开程序</summary>

``` python
import os
import subprocess
import sys
import xml.etree.ElementTree as ET

from concurrent.futures import ThreadPoolExecutor
from typing import Literal

import requests
import yaml
from tqdm import tqdm

rosdistro = "jazzy"
print("Fetching rosdep map...")
rosdep_map: dict[str, dict] = yaml.safe_load(
    requests.get(
        "https://ghfast.top/https://github.com/ros/rosdistro/raw/refs/heads/master/rosdep/base.yaml"
    ).text
)


def process_rosdep_package_list(
    rosdep_package_list: list[str] | dict[str, list[str]],
) -> str:
    system_package_name = ""
    if isinstance(rosdep_package_list, list):
        system_package_name = rosdep_package_list[0]
    else:
        for system in rosdep_package_list.keys():
            if rosdep_package_list[system]:
                system_package_name = rosdep_package_list[system][0]
    assert system_package_name
    return system_package_name


def rosdep_key_to_package_name(rosdep_key: str) -> str:
    package_name = ""
    if rosdep_key in rosdep_map.keys():
        current_package = rosdep_map[rosdep_key]
        if "arch" in current_package:
            package_name = process_rosdep_package_list(current_package["arch"])
        elif "ubuntu" in current_package:
            package_name = process_rosdep_package_list(current_package["ubuntu"])
        else:
            package_name = process_rosdep_package_list(
                current_package[list(current_package.keys())[0]]
            )
    else:
        original_name = rosdep_key.split("/")[-1]
        package_name = f"ros-{rosdistro}-" + original_name.replace("_", "-")
    if package_name == "python3-dev" or package_name == "python3":
        package_name = "python-3"
    if package_name.startswith(f"ros-{rosdistro}-python3-"):
        package_name = package_name[13 + len(rosdistro) :]
    if package_name.startswith("python3-"):
        package_name = package_name[8:]
    if package_name.endswith("-dev"):
        package_name = package_name[:-4]
    return package_name


class Package:
    version: str
    src: str
    check_update: str
    name: str
    dependencies: list[str]
    build_dependencies: list[str]
    description: str
    autobuild_type: str
    file_list_url: str
    package_xml_url: str
    package_info: ET.Element
    build_type: str

    def __init__(self, local_name: str, uri: str, version: str):
        semver = version.split("/")[-1]
        self.version = semver.replace("-", "+")
        version_prefix = "/".join(version.split("/")[:-1])
        self.src = f"git::commit=tags/{version_prefix}/{'${VER/+/-}'}::{uri}"
        repo = uri[19:-4]
        pattern = "/".join(version.split("/")[:-1]) + "/.*"
        self.check_update = f"github::repo={repo};pattern={pattern}"
        self.name = rosdep_key_to_package_name(local_name)
        self.file_list_url = (
            f"https://api.github.com/repos/{repo}/contents?ref={version}"
        )
        self.package_xml_url = (
            f"https://ghfast.top/{uri[:-4]}/raw/refs/tags/{version}/package.xml"
        )
        self.dependencies = [f"ros-{rosdistro}-ros-workspace"]
        self.build_dependencies = []

    def add_builddep(self, rosdep_key):
        package_name = rosdep_key_to_package_name(rosdep_key)
        if (
            package_name not in self.dependencies
            and package_name not in self.build_dependencies
        ):
            self.build_dependencies.append(package_name)

    def get_metadata(self):
        assert self.package_xml_url
        package_xml_string = requests.get(self.package_xml_url).text
        packge_info = ET.fromstring(package_xml_string)
        for elment in packge_info.findall("depend"):
            self.dependencies.append(rosdep_key_to_package_name(elment.text))
        for elment in packge_info.findall("exec_depend"):
            self.dependencies.append(rosdep_key_to_package_name(elment.text))
        for elment in packge_info.findall("build_export_depend"):
            self.dependencies.append(rosdep_key_to_package_name(elment.text))
        for elment in packge_info.findall("buildtool_export_depend"):
            self.dependencies.append(elment.text)
        for elment in packge_info.findall("buildtool_depend"):
            self.add_builddep(elment.text)
        for elment in packge_info.findall("build_depend"):
            self.add_builddep(elment.text)
        for elment in packge_info.findall("test_depend"):
            self.add_builddep(elment.text)
        raw_description = packge_info.findall("description")[0].text.splitlines()
        self.description = self.name
        for line in raw_description:
            clean_line = line.strip()
            if clean_line:
                self.description = clean_line
                break
        file_list = requests.get(self.file_list_url).json()
        has_pyproject = False
        has_setup_py = False
        has_cmake_lists = False
        for file in file_list:
            if file["name"] == "pyproject.toml":
                has_pyproject = True
            if file["name"] == "setup.py":
                has_setup_py = True
            if file["name"] == "CMakeLists.txt":
                has_cmake_lists = True
        if has_cmake_lists:
            self.build_type = "cmakeninja"
        elif has_pyproject:
            self.build_type = "pep517"
        elif has_setup_py:
            self.build_type = "python"
        else:
            self.build_type = "unknown"

    def generate_spec(self):
        return f"""VER={self.version}
SRCS="{self.src}"
CHKSUMS="SKIP"
CHKUPDATE="{self.check_update}"
"""

    def generate_defines(self):
        defines_content = f"""PKGNAME={self.name}
PKGSEC=ros
PKGDEP="{" ".join(self.dependencies)}"
"""
        if self.build_dependencies:
            defines_content += f'BUILDDEP="{" ".join(self.build_dependencies)}"'
        defines_content += f"""
PKGDES="{self.description}"

ABTYPE={self.build_type}
PREFIX="/opt/ros/{rosdistro}"
ABSPLITDBG=0
"""
        if self.build_type == "pep517" or self.build_type == "python":
            defines_content += """NOPYTHON2=1
ABHOST=noarch
"""
        if self.build_type == "cmakeninja":
            defines_content += f"""CMAKE_AFTER=(
    "-DCMAKE_PREFIX_PATH=/opt/ros/jazzy"
    "-DBUILD_TESTING=OFF"
)
"""
        defines_content += "NOSTATIC=0\n"
        return defines_content


def write_file(new_package: Package):
    dirname = f"runtime-ros/{new_package.name}"
    if not os.path.exists(dirname):
        try:
            new_package.get_metadata()
        except TypeError:
            print("API rate limit exceeded")
            return
        except requests.exceptions.SSLError:
            print("API rate limit exceeded")
            return
        except requests.exceptions.ConnectionError:
            print("API rate limit exceeded")
            return
        spec = new_package.generate_spec()
        defines = new_package.generate_defines()
        prepare = f"""export PYTHONPATH=/opt/ros/{rosdistro}/lib/python3.10/site-packages/
export PKG_CONFIG_PATH=/opt/ros/jazzy/lib/pkgconfig
. /opt/ros/jazzy/setup.sh
"""
        os.mkdir(dirname)
        os.mkdir(dirname + "/autobuild")
        open(dirname + "/spec", "w", encoding="utf-8").write(spec)
        open(dirname + "/autobuild/defines", "w", encoding="utf-8").write(defines)
        open(dirname + "/autobuild/prepare", "w", encoding="utf-8").write(prepare)


print("Scanning packages ...")
raw_package_type = dict[
    Literal["git"], dict[Literal["local-name", "uri", "version"], str]
]
try:
    rosinstall_generator_output = subprocess.run(
        ["rosinstall_generator", "desktop", "--rosdistro", rosdistro, "--deps"],
        capture_output=True,
        check=True,
    )
    raw_packages_str = rosinstall_generator_output.stdout
    raw_packages_data: raw_package_type = yaml.safe_load(raw_packages_str)
except subprocess.CalledProcessError as e:
    print(f"命令执行失败: {e.returncode}", file=sys.stderr)
    print(e.stderr)
except FileNotFoundError:
    print(f"找不到命令，rosinstall-generator 未安装", file=sys.stderr)
except yaml.YAMLError as e:
    print(f"返回值解析错误: {e}", file=sys.stderr)

print("Generating files...")
futures = []
if not os.path.exists("runtime-ros"):
    os.mkdir("runtime-ros")

# 多线程运行
with ThreadPoolExecutor(max_workers=10) as executor:
    for package in tqdm(raw_packages_data):
        current_data = package["git"]
        current_package = Package(
            current_data["local-name"], current_data["uri"], current_data["version"]
        )
        future = executor.submit(write_file, current_package)
        futures.append(future)
    for future in tqdm(futures):
        future.result()
```
<details>

## 手动处理依赖

虽然大多数依赖都使用脚本自动化完成了，还有一些依赖包名需要手动处理，例如`eigen3-dev`和`eigen-3`等包名不一致的，还有`importlib-*`等属于另一个包的一部分的情况，以及`ros-jazzy-ros-workspace`会产生循环依赖等等

这部分工作量不大且无法自动化，需要人工手动完成

## 处理架构不兼容的包

打包过程中遇到了一个特殊的包`Mimick`，它用于C的测试

它使用了[一些汇编代码](https://github.com/Snaipe/Mimick/tree/master/src/asm)，而且只有x86和arm，甚至[添加RV支持的PR](https://github.com/Snaipe/Mimick/pull/34)几年了也没有被上游合并。即使不考虑上游，学会写这么多架构的汇编也不是一件简单的事。所幸它只是一个用于测试的mock库，可以直接在不支持的架构上不依赖它并设置`-DBUILD_TESTING=OFF`，顺利解决
