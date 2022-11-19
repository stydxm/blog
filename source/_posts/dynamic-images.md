---
title: GIF之后，再无动态图片
date: 2022-03-26 13:15:00
updated: 2022-07-26 14:28:00
categories:
- [技术]
- [随想]
cover: https://s1.ax1x.com/2022/07/26/jzEahd.png
toc: true
---

24号看到GIF发明者去世的消息[2]，觉得GIF早已完成了它的历史使命。

但今天打开QQ空间，看到GIF的梗图，才意识到GIF仍然在被大量使用。

<!-- more -->

然后仔细一想，虽然GIF的分辨率低、帧率低、只有256色，虽然现在的web开发早已没有满屏闪烁的gif，没有雪碧图，虽然像谷歌苹果这样的巨头和各个开源组织都在不遗余力地推行更快或者更高压缩率的图形编码格式，比如webp和svg……但是，到现在除了gif，我们仍然没有第二种足够通行并且较轻量的动图格式，或许在将来较长时间里，也不会有。我们要么继续用gif，要么就只能用需要播放器的视频了。

**2022.7.26 update**

> 这篇文章原本发在B站动态和QQ空间里，今天搬到博客上的时候做了一点小修改，顺便查了一下除gif外现有的动态图片格式，然后发现了犯了个错误——Google的[WebP](https://developers.google.com/speed/webp)其实是支持动图的，只是webp的动图有多少，恐怕不好说。

支持动图的格式：

## [APNG](https://wiki.mozilla.org/APNG_Specification) - Animated Portable Network Graphics

> APNG is an extension of the PNG format, adding support for animated images. It is intended to be a replacement for simple animated images that have traditionally used the GIF format, while adding support for 24-bit images and 8-bit transparency. APNG is a simpler alternative to MNG, providing a spec suitable for the most common usage of animated images on the Internet. 

拓展自PNG，支持24bit位深度和8bit的透明图像

兼容PNG，第一帧为普通PNG，所以也可以作为普通PNG来解析

## [WebP](https://developers.google.com/speed/webp)

> The 14-bit dynamics for image size limit the maximum size of a WebP lossless image to 16384✕16384 pixels.

[官网上](https://developers.google.com/speed/webp/docs/webp_lossless_bitstream_specification?hl=en)说支持最高14bit，16K分辨率的动态图片

## [AVIF](https://aomediacodec.github.io/av1-avif/) - AV1 Image File Format

基于AV1的图片格式，支持动图

支持多种色彩空间，多种压缩率，多种位深度

## [MNG](http://www.libpng.org/pub/mng/) - Multiple-image Network Graphics

和APNG一样也拓展自PNG，而且得到了PNG的官方承认，但似乎目前兼容性还不如APNG



[1]: 封面图片：[Computer programmer Stephen Wilhite accepts his Webby Lifetime Achievement Award in 2013. (Webby Awards/AP)](https://www.washingtonpost.com/obituaries/2022/03/24/gif-creator-stephen-wilhite-dead/)  
[2]: [Stephen Wilhite, computer programmer who created the GIF, dies at 74](https://www.washingtonpost.com/obituaries/2022/03/24/gif-creator-stephen-wilhite-dead/)  
[3]: [Graphics Interchange Format](https://www.w3.org/Graphics/GIF/spec-gif87.txt)