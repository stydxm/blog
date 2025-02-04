---
title: 其他语言中的 password_hash()
toc: true
date: 2025-02-04 22:56:17
categories: 技术
cover:
---

PHP中有“一对函数”[^1]`password_hash()`和`password_verify()`，用于对密码的加密和验证

[^1]: PHP里密码哈希相关的还有另外三个，不过我想应该没啥人用？

若用其他语言重写PHP后端，通常不会影响到其他数据，但密码比较特殊：它被hash了。因此重写后若想保留用户数据就需要用相同的方式来验证，而且hash算法也是不可逆的，不可能使用其他算法重新编码。所以就需要想办法在其他语言中实现这两个函数

<!-- more -->

翻一下[手册](https://www.php.net/manual/en/function.password-hash.php)可以看到，这两个函数默认使用[`bcrypt`](https://en.wikipedia.org/wiki/Bcrypt)算法，那么就可以很轻松地在其他语言中找到第三方库的实现

|语言|库|
|:-:|:-:|
|Python|[bcrypt](https://pypi.org/project/bcrypt/)|
|JS|[node.bcrypt.js](https://www.npmjs.com/package/bcrypt)|
|Go|[crypto/bcrypt](https://pkg.go.dev/golang.org/x/crypto/bcrypt)|
