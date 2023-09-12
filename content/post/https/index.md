---
title: "HTTPS 分享"
date: 2023-09-12T00:23:00+08:00
tags: [https,cryptography]
toc: true
---

## 引言

这是2019年3月一次内部技术分享，当时在笔记上留下了一些大纲，分享时的PPT已经不见了，借此机会再写一遍分享内容，于自己是一次知识巩固，于朋友们也算得上是一次技术交流。

## 先聊聊密码学

谈到HTTPS，大家首先想到的是安全和加密通信。所以在开始HTTPS之旅前，有必要先聊一些密码学的内容。

### 古典密码

密码并不是一个很新潮的技术，我甚至怀疑从有人类文明开始，密码技术就诞生了。毕竟我们每个人心底里都藏着那没多不想让人知道的秘密😊

我们先来看看比较古老的密码。

#### 密码棒

密码棒是一种非常古老的密码，看着也有点像小时候的游戏。用一个纸条或者布条缠绕在有多个面（例如4个面）的棒子上，然后竖着在每个面上写字，写完后将纸条或者布条拉直，这时候得到的就是被打乱顺序的文字了。这也算得上一种密码吧！

{{< figureCupper
img="figure1-scytale.png"
caption="密码棒 参考[wiki](https://zh.wikipedia.org/wiki/密碼棒)"
command="Resize"
options="260x" >}}

间接的证据指出，最早提到密码棒的是一位公元前7世纪的希腊诗人阿尔基罗库斯，后来的希腊和罗马作家也在作品提到。

希腊历史学家普鲁塔克（Plutarch）曾写下密码棒的用法

wiki 上给了一个例子，按照上述方法处理 "Help me I am under attack" 这句话，加解密过程为：

加密过程：明文4个文字绕一圈，所以可以绕棒子5圈，故我们按照5个文字分一组，然后竖着读取每一列。（想象一下）：

```
H E L P M
E I A M U
N D E R A
T T A C K
```

得到密文：

```
HENTEIDTLAEAPMRCMUAK
```

解密过程则相反：将密文4个文字一组分组（每间隔3个文字取一个，到尾部后绕回即可。）

```
H E N T---------
E I D T---------
L A E A--------
P M R C---------
M U A K---------
```

得到明文：

```
HELPMEIAMUNDERATTACK
```

#### 凯撒密码

凯撒密码，即是人们常说的变换加密，根据苏维托尼乌斯的记载，凯撒曾用此方法对重要的军事信息进行加密。

{{< figureCupper
img="figure2-caesar-cipher.png"
caption="凯撒密码 参考[wiki](https://zh.wikipedia.org/wiki/凱撒密碼)"
command="Resize"
options="260x" >}}

对于字母文字而言，可以理解为按照一个固定的偏移量在字母表上将原始字母替换成目标字母。其变种也可以是预先设置好原始字母表和目标字母表的映射关系，然后做字母替换。

针对第一种情况，在偏移量为3时的一个例子

```
# 字母表
ABCDEFGHIJKLMNOPQRSTUVWXYZ
DEFGHIJKLMNOPQRSTUVWXYZABC

# 明文
THE QUICK BROWN FOX JUMPS OVER THE LAZY DOG

# 密文
WKH TXLFN EURZQ IRA MXPSV RYHU WKH ODCB GRJ
```





#### 维吉尼亚密码

#### 谜码机

#### 一次性密码本

### 现代密码的工具箱

#### 对称加密

#### 非对称加密（公钥加密）

#### 单向散列函数

#### 消息确认码

#### 数字签名

#### 伪随机数生成器


## 为什么是HTTPS

### HTTP演进历史

### 什么是HTTPS

### 如何实现安全传输


## 有关数字证书

### 数字证书

### 验证过程

### 证书链

### 证书获取方式

### Let's Encrypt

## 参考

* [《图解密码技术》](https://www.ituring.com.cn/book/1737)
* [《HTTPS权威指南》](https://www.ituring.com.cn/book/1734) 
* Let's Encrypt [官方网站](https://letsencrypt.org/zh-cn/)

再多的参考资料都不如自己动手实践，共勉！


