---
title: 使用SSH作为跳板机
toc: true
date: 2025-03-24 23:36:55
categories: 技术
cover:
---

在以前，云服务商提供的国内服务器带宽都很有限或非常昂贵，前段时间阿里云发布低价的200M带宽的轻量应用服务器，使得带宽成本大大降低

一些在已有机器上的服务迁移麻烦，或者因为储存/算力的原因无法迁移，那么就自然而然地想到可以用大带宽机器中转，同时享受到大带宽和高算力/大储存

<!--more-->

# 要求
首先原有服务器 `A` 和200M的服务器 `B` 要在同一可用区，这样才可以通过千兆内网传输数据

> 这里的A不一定是轻量应用服务器，即使没有公网IP，只要与B内网相通都可以用这个方法，比如ECS、裸金属服务器，甚至开了SSH的serverless也行
>
> B必须要能使用SSH登陆，如果不能的话需要考虑其他方式进行端口转发

# 配置SSH密钥登陆

密码连接需要手动介入，为了能在脚本中自动化运行，需要使用密钥或者证书登陆

这里不写教程了，如有需要可以参考[WangDoc上的教程](https://wangdoc.com/ssh/key)，需要将客户端的密钥添加到A和B上

# 获取连接地址

打开阿里云的控制台，在基本信息的位置获取服务器A的内网地址（图中`主私网IP`）和服务器B的公网地址（图中`公网IP`）

![](https://s21.ax1x.com/2025/03/24/pEB6PIJ.png)

如果有防火墙、黑白名单等需要注意，客户端需要能连接到B，同时B要能连接到A

因为到A的流量会从B中转，不经过A的公网地址，所以客户端能否连接到A并不重要

# 使用SSH协议的服务

对于使用SSH连接的服务，比如SSH本身，或者sftp scp rsync等，可以使用ProxyJump来将B作为跳板机连接A

## 在命令行中测试连接

先打开任意一个终端，输入命令测试到A的连接，注意将参数替换为自己的

``` bash
ssh -J 用户名B@B的内网IP 用户名A@A的公网IP
```

如果使用的不是标准的22端口，则是

``` bash
ssh -J 用户名B@B的内网IP:B的端口 用户名A@A的公网IP -p A的端口
```

如果正常的话，这时应该会进入A的终端

## 将连接信息写入到配置文件

在`~/.ssh/config`中添加如下内容（这个文件不存在就手动新建）

``` ssh
Host ServerB
  HostName B的公网IP
  User B的用户名

Host ServerA
  HostName A的内网IP
  User A的用户名
  ProxyJump ServerB
```

说明：

- 这里的Host后面的名字是主机名，可以自己改

- ProxyJump后面的内容也可以直接写IP的，但建议还是把B单写出来，这样在连接服务器的时候可以直接使用`ssh ServerB`

- 如果端口不是标准的22，可以在对应服务器的配置下面加`Port 端口号`

- 用户名与当前登陆的用户一致，就可以不写User字段，后面都同理

- 如有需要可以“以此类推”地串联下去，如果使用命令则是`ssh 用户名3@服务器3的内网IP -J 用户名1@服务器1的公网IP,用户名2@服务器2的内网IP`，连接顺序是1->2->3

## 使用

完成配置后，在使用rsync和scp等工具时可以用ServerA（或者自己设的其他名字）来代替服务器A的地址

例如使用rsync同步本地和远程的`/home/user/`目录：

```bash
rsync -a /home/user/ ServerA:/home/user/
```

# 非SSH协议的服务

比如http等不使用SSH的协议，那么就无法通过ProxyJump的方式来连接，需要通过服务器B的SSH对A上的服务进行端口转发

> 注意这里的HTTP服务指控制面板等，并不适用于需要公开访问的网站，那种情况请使用Nginx等软件进行反代

> 其实这种方法对于SSH协议也生效，只是步入ProxyJump方便

## 命令行中使用

这里服务器A只需要填IP和端口，连接的其他信息如用户名等在下一步连接时提供

本地端口可以任意填，可以和服务器A上业务的端口不一样，但注意不要与本地其他服务冲突

```bash
ssh -L -N -f 本地端口:服务器A的内网IP:用户名A@服务器A的服务端口 用户名B@B的公网IP
```

> 如果没有写`-f`参数，则终端会被阻塞没有响应，这是正常的，退出则需要按`Ctrl+C`

连接上后，就可以使用127.0.0.1和本地端口来连接服务器A上的服务了

比如http服务，就可以在浏览器打开`http://127.0.0.1:本地端口`

## 写入配置文件

本地转发也能在写入到配置文件中 ~~虽然不太常见~~

``` ssh
Host ServerB
  HostName B的公网IP
  User B的用户名
  LocalForward 本地端口 A的内网IP:A的端口
```

# 参考

[SSH to remote hosts through a proxy or bastion with ProxyJump](https://www.redhat.com/en/blog/ssh-proxy-bastion-proxyjump)

[SSH 端口转发](https://wangdoc.com/ssh/port-forwarding)