---
title: Windows字体渲染优化
date: 2019-03-17 21:18:25
tags: 技术
urlname: improve-windows-font-rendering
---

从2k屏的Surface book换到24寸的1080p Dell 屏，Windows 默认的字体渲染几乎不能忍。因为我很穷，买不起高分辨率的显示器，只好研究一些奇怪的方式来优化这电脑的使用体验。<!-- more -->

## noMeiryoUI

[noMeiryoUI](https://github.com/Tatsu-syo/noMeiryoUI/) 是我第一个尝试的方案。它可以在不修改系统文件的前提下修改系统的字体。按照网上的推荐，我将字体改成了[更纱黑体](https://github.com/be5invis/Sarasa-Gothic)。效果还是有一点的，但是 Windows 字体最大的问题——笔画的锯齿非常明显，并没有得到改善。

## Mactype

在沉寂了许久以后，最新的 [MacType 2018.1-beta5](https://github.com/snowie2000/mactype/releases/tag/2018.1-beta5) 已经大大改善了兼容性问题。现在 Mactype 已经可以支持包括 Chrome 在内的大多数 DirectWrite 程序。

![Snipaste_2019-03-17_21-52-33.png](https://i.loli.net/2019/03/17/5c8e515376ea6.png)

Mactype 的加载方式，我选择的服务加载。第一次使用的时候可以使用托盘加载，没有问题再切换成其他模式。使用托盘加载时我有时会遇到 Chrome 不加载 Mactype 的问题，不知道是不是 bug。

Mactype 的配置推荐 [James 大佬](https://gist.github.com/Jamesits/6a58a1b08d5cd09a94a02f30ddaf0e13)的。我在他的配置上又做了一些修改。从L324到L282我全部注释了，因为加载这些进程在我这里没有遇到什么奇怪的问题，而且文件管理器毕竟还是经常要用到的。我也没有把 Chrome 中的雅黑替换成思源黑体，因为我也没有遇到奇怪的 bug。然后，在 Chrome 中用 XStyle 之类的插件加载这个 [CSS](https://userstyles.org/styles/159549/theme)。然后，[ExcludeSub] 中加入了 origin 和 uplay 的进程，这两个程序替换字体后图标都变成了方框。

### [Exclude] 呢？这个配置有什么用？

答案就是没有用，作者在 [Issue](https://github.com/snowie2000/mactype/issues/332#issuecomment-313572706) 中解释过了。

> [Exclude] - does not exist.

### 许多配置不生效？

很有可能是文件编码的问题。从 GitHub 上下载的文件默认是 UTF-8 编码，但是 Mactype 识别的是 GB 2312 编码。需要你自己用文本编辑器转换一下。

### 效果

![Snipaste_2019-03-17_22-15-04.png](https://i.loli.net/2019/03/17/5c8e567035bd6.png)

Mactype 的渲染策略大概是，当分辨率不足以清晰的渲染字体的时候，用一团灰色的东西来渲染对应的笔画，说的简单点就是抗锯齿。对于天天看中文的人来说，不用看清每一个笔画就能看懂对应的字，而且这应该是分辨率不足的情况下最大程度模仿印刷效果的方式，比微软选择的 Hinting 好很多（大概）。