---
title: 终于用上MBP了
date: 2019-09-15 20:21:03
description: ["公司给发了个MacBook Pro 然后就基本告别Manjaro了,这里介绍下使用osx的一些体验"]
toc: true
author: tabris
# 图片推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如:http://xxx.com/xxx.jpg
img:
# 如果top值为true,则会是首页推荐文章
top: false
# 如果要对文章设置阅读验证密码的话,就可以在设置password的值,该值必须是用SHA256加密后的密码,防止被他人识破
password:
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
summary:
categories: 实用技能
tags:
  - osx
---
[toc]

# osx 真香!

> 公司给发了个 MacBook Pro 然后就基本告别 Manjaro 了,
>
> 这里介绍下使用 osx 的一些体验
>

> [Mac tips - 打开【键盘重复按键】功能](https://blog.csdn.net/huhuijun123/article/details/84815267)
>
> 很多 App 在 Mac 下长按某个键时只会触发一次。 比如在 Sublime Text 下， 用 VIM 模式来 操作时， 长按 「J」 时， 只会按下跳一行。 但是奇怪开了中文输入法后又可以一直往下跳。
> 其实我们是可以用下面的命令来重新默认打开这个功能。
>
> ```shell
> defaults write -g ApplePressAndHoldEnabled -bool false
> ```
>
> 注销并重新登录系统使其更改生效。
> 如果需要恢复长按键盘可以重音字符或非英文字符的功能，请打开终端窗口，运行以下命令：
>
> ```shell
> defaults delete -g ApplePressAndHoldEnabled
> ```
>

## 软件篇

### GUI 软件

**vscode**: 开源的编辑器, 体验很棒

~~**jetbrains tool box**: 管理 jetbrains 哪些 ide 和工程的,推荐~~

**Chrome**: 浏览器, 依赖于它的很多插件, 换不了,  太费内存了...

**Typora**: Markdown 编辑器, 单页 实时预览,很 nice

~~**proxifer**: 改代理用的软件.~~

**Beyound Compare**: 文件比较工具, 本来不想用的, Git 的话 就直接 `git diff` 了, 但是公司好多代码还在 `svn` 上.

**onedrive**: 微软家的网盘, 跨平台同步的工具,

~~**navicat**: 操作数据库的, 超级好用.(收费软件，后来用 sequal pro 代替)~~

~~**markText**: Markdown 编辑器, 挺好用的.~~

**[Fliqlo](https://fliqlo.com/)**: 好看翻页时钟屏保。

---

> mac 独有分割线
>

**iina**: 看视频的软件,大多格式的都能放

**itsycal**: 在 menu bar 上的时间点一下会出现日历

~~**cheatsheet**: 长按 `command` 出现当前应用所有快捷键, 用 `键指如飞/FlyKey` 代替了~~

**iglance**: 在 Mac menu bar 上显示 CPU 内存网速电池等信息的小工具

~~**Alfred4**: 启动器, 比自带的好用得多,  但高端功能要花钱.~~

**utools**: 一款跨平台的启动器，插件免费，更加简单易用。

**cakebrew**: homebrew 的可视化版, 还挺好用, 但是有的软件找不到..

**karabiner**: 键盘映射工具

**keycastr**:显示键盘按键的软件

**GetPlainText**: 复制时删除样式.

**bartender 4**: 整理 menu bar 的工具.

**ForkLift**: finder 的替代品,

**Lemon**: 柠檬清理.

**Get Plain Text**: 复制粘贴的时候清除格式

**SwitchResX**: 快速修改屏幕分辨率的 Mac 软件

**[Better And Better 2.0](https://www.better365.cn/bab2.html)**: Better And Better 2.0 将强大功能与优秀人机交互结合提升到一个崭新的高度。全面提升 Mac 触控板、鼠标、键盘使用，数百种动作手势、绘图手势与预设、脚本、快捷键完美协作，为你带来无与伦比的 Mac 操作体验。

**[超级右键](https://www.better365.cn/irightmouse.html)**: 超级右键以优秀设计与丰富功能为 Mac 带来绝佳的使用体验，众多功能与右键融为一体，深得人心的设计，让你即刻进入高效的 Mac 使用体验，快、高效、便捷，效果出奇的好

**[ishot](https://www.better365.cn/ishot.html)**: iShot 堪称 macOS 上功能最为全面的截图、录屏工具，
截图、长截图、多窗口截图、延时截图、标注、贴图、取色、录屏......

**[自动切换输入法](https://www.better365.cn/AutoSwitchInput.html)**: 在自动切换输入法内，提前设置每个 App 对应的输入法，切换至该 App 时，将为您自动切换至为他设定好的输入法。

**[键指如飞/FlyKey](https://www.better365.cn/FlyKey.html)**: 键指如飞默认使用双击 Command 显示当前 App 的所有快捷键

**[rectangle](https://rectangleapp.com/)**: 桌面窗口管理。

**[pock](https://pock.dev/)**: 在触控栏中显示 macOS 程序坞

![](https://camo.githubusercontent.com/401d36fc151b85b5c001acb6c026c4c33c86aea9949a73bd2bbab811678e9a77/68747470733a2f2f706f636b2e6465762f6173736574732f696d672f707265766965772f706f636b5f776964676574732e706e67)

**[copyless 2](https://copyless.net/)**: 剪贴板管理器. 最多可以存储 1000 个最新剪辑

**[Mos](https://github.com/Caldis/Mos)**: 一个用于在 MacOS 上平滑你的鼠标滚动效果的小工具, 让你的滚轮爽如触控板。

**[Amphetamine](https://apps.apple.com/cn/app/amphetamine/id937984704?mt=12)**: 防睡眠软件。

---

### Terminal 软件

**iTerm2**: mac 下的终端模拟软件,**其实是 GUI 软件的,故意放在这里**

~~**thefuck**: 帮忙修正手误导致的错误命令~~

**tmux**: 终端复用工具,

**neovim**: 从 VIM 上 fork 来的, (没怎么用过 vim, 无法做出比较,

**ranger**: 终端下的文件管理器,配置后能预览图片,显示压缩文件信息等, 加上类 vi 的操作方式,很奈斯

~~**wtfutil**: 命令行下的仪表盘工具, 插件丰富,同时也可以自己开发提交 pr, 比较推荐~~

**homebrew**: mac 的软件包管理器, 一般好用吧, 用过 pacman 感觉其他的都不太行

**glances**: python 实现的高级 top 工具, 之前有时候会崩，崩的时候用 `htop` 代替

**docker**: mac 的 docker 感觉和 Linux 的不太一样 会有个应用程序在启动器里面..

**ripgrep**: 搜索工具, 快

**lsd**: ls 的替代品, rust 写的 好看又快

**nvm**: Node Version Manager, node 版本管理工具, 好用

**whistle**: 代理配置软件

**mycli**: 好用的 MySQL 客户端