---
title: 传输巨量数据：卡车拉硬盘，还真不是个梗
date: 2022-11-19 13:56:34
categories: 技术
cover: https://s1.ax1x.com/2022/11/19/zKSXUU.jpg
toc: true
excerpt: 前几天看到一张表情包，大意是可以用卡车转移数据到aws<br>本来以为是玩梗，没想到去搜了一下之后发现aws真的有这样的服务
---

# 前言

> Never underestimate the bandwidth of a station wagon full of tapes hurtling down the highway. -- [Andrew S. Tanenbaum](https://en.wikipedia.org/wiki/Andrew_S._Tanenbaum)[^1]

前几天看到一张表情包，大意是可以用卡车转移数据到aws  
本来以为是玩梗，没想到去搜了一下之后发现aws真的有这样的服务，叫做[AWS Snowmobile](https://aws.amazon.com/cn/snowmobile/)

![](https://s1.ax1x.com/2022/11/19/zKSXUU.jpg)  

查资料的时候发现已经有人给直接移动储存介质的传输方式取了个名字——Sneakernet，中文叫球鞋网络或者跑腿网络[^2]，于是就想写写关于大量数据传输的内容  
在云计算时代，如果企业要从传统的机房迁移到云服务商，势必要传输大量的数据；当数据达到一定量的时候，从企业自己服务器的硬盘传到云服务商的储存中去也是很困难的事，云服务器为了方便客户迁移，也给出了各种迁移方案  
查资料的时候也发现aws已经推出了很多sneakernet的服务，所以下面提到云服务商的时候主要以aws为例 ~~我看到snowmobile才想要写这篇的~~

# 网络
通过网络传输数据是最常规、最简便的方法，对于绝大多数个人和企业的非业务数据，直接通过网络传输数据方便快捷，必然是首选  
但是对于数据规模较大的情况，即使不考虑大量数据上传的网络带宽费用，即使以千兆网络的理论速率，每周也只能上传约75T的数据  
这样的速率对于数据量稍大的情况来说——比如企业的数据库、科研数据等——意味着需要付出很长的时间和不小的流量开支，而且耗时太长

# 运输储存介质
当数据量大到网络传输数据的耗时不可接受时，就需要使用sneakernet传输数据  
以aws为例，他们已经有很多[^3]专门用于数据上云的服务了，其中的[Snow系列](https://aws.amazon.com/cn/snow/)就是sneakernet的应用

## 直接邮寄储存介质
> Many years ago, professor Andy Tanenbaum wrote the following:
>> Never underestimate the bandwidth of a station wagon full of tapes hurtling down the highway.  
>
> Since station wagons and tapes are both on the verge of obsolescence, others have updated this nugget of wisdom to reference DVDs and Boeing 747s.[^4]  

### AWS Import/Export
早在2009年3月，阿里云和azure都还不存在的时候，aws就[推出了](https://aws.amazon.com/blogs/aws/send-us-that-data/)这个服务  
客户可以把小于50磅、8U[^4]的储存设备寄给亚马逊，亚马逊会把里面的文件导入S3  
不过目前这个服务已经停止了，博客中留的[产品页面](http://aws.amazon.com/importexport)现在会被重定向到snowball，大概是因为已经有其他更成熟、更细分的服务可以上云了吧  
~~当年[用来计算耗时的计算器](http://awsimportexport.s3.amazonaws.com/aws-import-export-calculator.html)居然现在还能用~~  
虽然由于物理接口速度的限制（当时的USB 2.0和eSATA都还不到千兆），每周只能传输大概40-50TB的数据，现在看起来速度并不快，但对比当时的网速这并不慢，不妨是一种成功的尝试  

### 其他
清华也运过有700TB实验数据的磁带[^5]  

## 将数据装载在专用设备中邮寄

### AWS Import/Export Snowball
> We launched the first-generation AWS Snowball service way back in 2009. As I wrote at the time, “Hard drives are getting bigger more rapidly than internet connections are getting faster.” I believe that remains the case today. In fact, the rapid rise in Big Data applications, the emergence of global sensor networks, and the “keep it all just in case we can extract more value later” mindset have made the situation even more dire.[^6]  

2015年10月亚马逊[推出了](https://aws.amazon.com/blogs/aws/aws-importexport-snowball-transfer-1-petabyte-per-week-using-amazon-owned-storage-appliances/)Snowball，速度提升到了每周1PB[^6]，而且有了实体的设备

![](https://s1.ax1x.com/2022/11/19/zKJ2qI.png)

这个snowball设备就是个箱子，有电源、网口和一块屏幕，我猜测内部是类似普通开发板的架构和一些硬盘，最多可以储存50TB数据[^6]  
它的使用方法变成了客户在内网里向snowball写入数据，然后寄给亚马逊，亚马逊会进行导入  
做出这样改变的原因我猜测是这样标准化的设备在亚马逊的数据中心里可以自动化操作，而以前五花八门的设备显然做不到；而且manifest由程序自动生成，方便使用的同时也减少了错误  
aws另外发布过[Snowball Edge](https://aws.amazon.com/blogs/aws/aws-snowball-edge-more-storage-local-endpoints-lambda-functions/)、[Snowcone](https://aws.amazon.com/blogs/aws/introducing-aws-snowcone-small-lightweight-edge-storage-and-processing/)这些设备，只是介质和容量有所不同，其他也都大同小异
![](https://s1.ax1x.com/2022/11/19/zKg9p9.png)
![](https://s1.ax1x.com/2022/11/19/zKgkm6.png)

### Azure Data Box
azure也有类似服务，叫[Azure Data Box](https://azure.microsoft.com/zh-cn/products/databox/data/)，有40TB的Data Box Disk、100TB的Data Box、1PB的Data Box Heavy三种，这里讲讲“标准款”的Data Box  

![](https://s1.ax1x.com/2022/11/19/zKBrJ1.png)

它有一个电口两个光口，共三个万兆接口

### Transfer Appliance
GCP也有这样的设备[Transfer Appliance](https://cloud.google.com/transfer-appliance)  
Google只说了有40TB的TA40和300TB的TA300两种型号，但甚至连一张照片都没有发  
~~属实保密程度高~~  

# 上门收货
## AWS Snowmobile
这就是我当时看到的那个卡车拉硬盘的服务，aws更是把卡车开到了[发布会](https://www.youtube.com/watch?v=8vQmTZTq7nw)上  
![](https://s1.ax1x.com/2022/11/19/zKEF4P.png)  
确切地说，snowmobile并不包含卡车，而是一个集装箱  
它可以储存100PB数据，有共计1T的网络带宽，可以在几周内传输EB级的数据
~~钱给到位，亚马逊甚至可以给你安排发电机、安排车辆护送、安排安保人员哦~~

[^1]: Tanenbaum, Andrew S. (1989). [Computer Networks](https://archive.org/details/computernetworks02tane/page/57). New Jersey: Prentice-Hall. p. 57. ISBN 0-13-166836-6.  
[^2]: [Sneakernet](https://en.wikipedia.org/wiki/Sneakernet)  
[^3]: [Migration & Transfer on AWS](https://aws.amazon.com/cn/products/migration-and-transfer/)  
[^4]: [AWS Import/Export: Ship Us That Disk!](https://aws.amazon.com/blogs/aws/send-us-that-data/)  
[^5]: [金枪鱼之夜：空运磁带的 PB 级实验数据传输](https://tuna.moe/event/2022/lto-practice/)  
[^6]: [AWS Import/Export Snowball – Transfer 1 Petabyte Per Week Using Amazon-Owned Storage Appliances](https://aws.amazon.com/blogs/aws/aws-importexport-snowball-transfer-1-petabyte-per-week-using-amazon-owned-storage-appliances/)  
[^7]: [AWS Snowmobile – Move Exabytes of Data to the Cloud in Weeks](https://aws.amazon.com/blogs/aws/aws-snowmobile-move-exabytes-of-data-to-the-cloud-in-weeks/)  
[^8]: [AWS Snowball Edge – More Storage, Local Endpoints, Lambda Functions](https://aws.amazon.com/blogs/aws/aws-snowball-edge-more-storage-local-endpoints-lambda-functions/)  
[^9]: [Azure Data Box Datasheet](https://azure.microsoft.com/zh-cn/resources/azure-data-box-heavy-datasheet/)
[^10]: [Transfer Appliance Specifications](https://cloud.google.com/transfer-appliance/docs/4.0/specifications)