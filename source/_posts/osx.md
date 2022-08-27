---
title: 终于用上MBP了
date: 2019-09-15 20:21:03
updated: 2022-05-02 14:54:11
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
categories:
  - 学习
  - 工具
tags:
  - osx
---
[toc]

# MacOS 真香!

> 公司给发了个 MacBook Pro 然后就基本告别 Manjaro 了,
>
> 这里介绍下使用 osx 的一些体验
>

## 安装包管理器

### **[homebrew](https://brew.sh/index_zh-cn)**

 mac 的软件包管理器, 一般好用吧, 用过 pacman 感觉其他的都不太行

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

```bash
# 换清华源 https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git"
for tap in core cask{,-fonts,-drivers,-versions} command-not-found; do
    brew tap --custom-remote --force-auto-update "homebrew/${tap}" "https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-${tap}.git"
done
brew update
```

### 基础配置

#### 键盘重复按键

> [Mac tips - 打开【键盘重复按键】功能](https://blog.csdn.net/huhuijun123/article/details/84815267)
>
> 很多 App 在 Mac 下长按某个键时只会触发一次。 比如在 Sublime Text 下， 用 VIM 模式来 操作时， 长按 「J」 时， 只会按下跳一行。 但是奇怪开了中文输入法后又可以一直往下跳。
> 其实我们是可以用下面的命令来重新默认打开这个功能。
>
> ```bash
> defaults write -g ApplePressAndHoldEnabled -bool false
> ```
>
> 注销并重新登录系统使其更改生效。
> 如果需要恢复长按键盘可以重音字符或非英文字符的功能，请打开终端窗口，运行以下命令：
>
> ```bash
> defaults delete -g ApplePressAndHoldEnabled
> ```

#### 字体

> 安装`JetBrains Mono Nerd Font`字体
>
> ```bash
> brew tap homebrew/cask-fonts
> brew install font-jetbrains-mono-nerd-font
> ```

## 软件篇

### GUI 软件

**[vscode](https://code.visualstudio.com/)**: 开源的编辑器, 体验很棒

> `brew install --cask visual-studio-code`

**[Chrome](https://www.google.com/intl/zh-CN/chrome/)**: 浏览器, 依赖于它的很多插件, 换不了,  太费内存了...

> `brew install google-chrome`
> 还需要装很多插件，TODO

**[Typora](https://typora.io/)**: Markdown 编辑器, 单页 实时预览,很 nice

> `brew install --cask typora`

**onedrive**: 微软家的网盘, 跨平台同步的工具,

~~**[markText](https://marktext.app/)**: Markdown 编辑器, 挺好用的.~~

**[Fliqlo](https://fliqlo.com/)**: 好看翻页时钟屏保。

> `brew install --cask fliqlo`

**[滴答清单](https://dida365.com/)**：TODO 类软件，全平台支持了，挺好用的。

> `brew install --cask ticktick`

**[SourceTree](https://www.sourcetreeapp.com/)**: 免费好用的git客户端软件。

> `brew install --cask sourcetree`

**[NetNewsWire](https://netnewswire.com/)**: RSS阅读器

> `brew install --cask netnewswire`

**[OBS](https://obsproject.com/zh-cn)**: 免费且开源的用于视频录制以及直播串流的软件。

> `brew install --cask obs`

---

> mac 独有分割线
>

**[iina](https://iina.io/)**: 看视频的软件,大多格式的都能放

> `brew install --cask iina`

**[itsycal](https://www.mowglii.com/itsycal/)**: 在 menu bar 上的时间点一下会出现日历

> `brew install --cask itsycal`

**[utools](https://u.tools/)**: 一款跨平台的启动器，插件免费，更加简单易用。

> `brew install --cask utools`

> 还需要装很多插件，TODO

~~**[cakebrew](https://www.cakebrew.com/)**: homebrew 的可视化版, 还挺好用, 但是有的软件找不到..~~

**[karabiner](https://karabiner-elements.pqrs.org/)**: 键盘映射工具

> `brew install --cask karabiner-elements`

**[keycastr](https://github.com/keycastr/keycastr)**:显示键盘按键的软件

> `brew install --cask keycastr`

**[GetPlainText](https://apps.apple.com/us/app/get-plain-text/id508368068)**: 复制时删除样式.

**[bartender 4](https://www.macbartender.com/Bartender4/)**: 整理 menu bar 的工具.

> `brew install bartender --cask`

**[Lemon](https://lemon.qq.com/)**: 柠檬清理.

> `brew install tencent-lemon --cask`

**[SwitchResX](https://www.madrau.com/srx_download/download.html)**: 快速修改屏幕分辨率的 Mac 软件

> `brew install switchresx`

**[Better And Better 2.0](https://www.better365.cn/bab2.html)**: Better And Better 2.0 将强大功能与优秀人机交互结合提升到一个崭新的高度。全面提升 Mac 触控板、鼠标、键盘使用，数百种动作手势、绘图手势与预设、脚本、快捷键完美协作，为你带来无与伦比的 Mac 操作体验。

**[超级右键](https://www.better365.cn/irightmouse.html)**: 超级右键以优秀设计与丰富功能为 Mac 带来绝佳的使用体验，众多功能与右键融为一体，深得人心的设计，让你即刻进入高效的 Mac 使用体验，快、高效、便捷，效果出奇的好

~~**[ishot](https://www.better365.cn/ishot.html)**: iShot 堪称 macOS 上功能最为全面的截图、录屏工具，截图、长截图、多窗口截图、延时截图、标注、贴图、取色、录屏......~~

**[snipaste](https://snipaste.com/)**: 开源的多平台截图软件。

> `brew install snipaste`

**[自动切换输入法](https://www.better365.cn/AutoSwitchInput.html)**: 在自动切换输入法内，提前设置每个 App 对应的输入法，切换至该 App 时，将为您自动切换至为他设定好的输入法。

**[键指如飞/FlyKey](https://www.better365.cn/FlyKey.html)**: 键指如飞默认使用双击 Command 显示当前 App 的所有快捷键

**[rectangle](https://rectangleapp.com/)**: 桌面窗口管理。

> `brew install --cask rectangle`

~~**[copyless 2](https://copyless.net/)**: 剪贴板管理器. 最多可以存储 1000 个最新剪辑~~

**[Mos](https://github.com/Caldis/Mos)**: 一个用于在 MacOS 上平滑你的鼠标滚动效果的小工具, 让你的滚轮爽如触控板。

> `brew install --cask mos`

**[Amphetamine](https://apps.apple.com/cn/app/amphetamine/id937984704?mt=12)**: 防睡眠软件。

~~**[TopNotch](https://topnotch.app/)**: 针对2021款MBP隐藏刘海的软件。> `brew install --cask topnotch`~~

**[Only Switch](https://jacklandrin.github.io/macos%20app/2021/12/01/onlyswitch.html)**: OnlySwitch 是一个菜单栏的工具箱，提供很多快捷的功能，例如隐藏桌面图标，黑暗模式和隐藏新 MacBook Pro 的丑陋缺口。开关显示在您的状态栏上。目前支持以下功能：隐藏桌面 黑暗模式 屏幕保护程序 自动隐藏Dock AirPods 蓝牙 Xcode 缓存 自动隐藏菜单栏 显示隐藏文件 广播电台 保持清醒 清空垃圾桶 清空粘贴板 静音 隐藏留海等。

> `brew install only-switch`

**[Parallels Desktop](https://www.parallels.cn/)**: Mac 上的虚拟机软件

> `brew install --cask parallels`

**[PD Runner](https://github.com/lihaoyun6/PD-Runner)**: 适用于Parallels Desktop的启动器, 可无视试用期限强制启动客户机

> ~~`brew install pd-runner`~~
>
> github 已经被封了，brew 安装不了。 点击[这个](https://github.com/lihaoyun6/BigSur-icons/files/7968303/pdlatest.zip)下载吧
> 来源：[https://github.com/lihaoyun6/BigSur-icons/issues/6#issuecomment-1025392331](https://github.com/lihaoyun6/BigSur-icons/issues/6#issuecomment-1025392331)


---

### Terminal 软件

**[homebrew](https://brew.sh/index_zh-cn)**: mac 的软件包管理器, 一般好用吧, 用过 pacman 感觉其他的都不太行

> `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"`

```bash
# 换清华源 https://mirrors.tuna.tsinghua.edu.cn/help/homebrew/
export HOMEBREW_CORE_GIT_REMOTE="https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git"
for tap in core cask{,-fonts,-drivers,-versions} command-not-found; do
    brew tap --custom-remote --force-auto-update "homebrew/${tap}" "https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-${tap}.git"
done
brew update
```

**[alacritty](https://github.com/alacritty/alacritty/)**: 跨平台、高性能的终端模拟器。vim下大文件体验很棒。

> `brew install --cask alacritty`

**[iTerm2](https://iterm2.com/)**: mac 下的终端模拟软件,**其实是 GUI 软件的,故意放在这里**

> `brew install --cask iterm2`

~~**thefuck**: 帮忙修正手误导致的错误命令~~

> `brew install thefuck`

**tmux**: 终端复用工具,

> `brew install tmux`

> [很棒的tmux配置](https://github.com/gpakosz/.tmux)

**neovim**: 从 VIM 上 fork 来的, (没怎么用过 vim, 无法做出比较,

> `brew install neovim`

**ranger**: 终端下的文件管理器,配置后能预览图片,显示压缩文件信息等, 加上类 vi 的操作方式,很奈斯

> `brew install ranger` or `pip3 install ranger-fm`

**docker**: mac 的 docker 感觉和 Linux 的不太一样 会有个应用程序在启动器里面..

> `brew install docker`

**ripgrep**: 搜索工具, 快

> `brew install ripgrep`

**lsd**: ls 的替代品, rust 写的 好看又快

> `brew install lsd`

**whistle**: 代理配置软件

> `brew install whistle` or `npm install -g whistle`

**mycli**: 好用的 MySQL 客户端

> `brew install mycli` or `pip3 install mycli`


### 其他配置

#### 快速查看(空格一键预览)

Quick Look 是 macOS 最方便的功能之一，因为它可以从任何 Finder 窗口立即访问。 只需点击空格键，您就会立即看到当前选择的任何文件的大预览。

> [macOS 快速预览与插件马克 - 乌图米的文章 - 知乎](https://zhuanlan.zhihu.com/p/105834572)

> `brew cask install qlcolorcode qlstephen qlmarkdown quicklook-json qlimagesize webpquicklook qlvideo provisionql quicklookapk --cask`

#### 输入法

[鼠须管](https://rime.im/)是一款 macOS 下的开源输入法，支持拼音、注音、仓颉、五笔等输入方案，它底层是 RIME/中州韵输入法引擎，被誉为神级输入法。

经过配置之后 会很好用

[鼠须管输入法介绍 - 纤夫张的文章 - 知乎](https://zhuanlan.zhihu.com/p/82476313)

> `brew install squirrel --cask`

> 我用的配置
> 1. [ssnhd/rime-Rime Squirrel 鼠须管配置文件（朙月拼音、小鹤双拼、自然码双拼）](https://github.com/ssnhd/rime)
> 2. [禁用 Squirrel 英文模式，使用左侧 Shift 切换中英](https://github.com/rime/squirrel/wiki/%E7%A6%81%E7%94%A8-Squirrel-%E8%8B%B1%E6%96%87%E6%A8%A1%E5%BC%8F%EF%BC%8C%E4%BD%BF%E7%94%A8%E5%B7%A6%E4%BE%A7-Shift-%E5%88%87%E6%8D%A2%E4%B8%AD%E8%8B%B1)
>
> 我是在1的基础上参考2禁用了英文模式.
> > 用了自动切换输入法的功能, 英文用自带的输入既可. 鼠须管只用做中文输入.


#### 系统设置

##### 命令设置

> https://blog.csdn.net/lovechris00/article/details/113280758
> https://macos-defaults.com/
> https://github.com/kevinSuttle/macOS-Defaults <- 这个最全。

```bash
# Finder 的退出按钮
defaults write com.apple.finder "QuitMenuItem" -bool "true" && killall Finder

# 允许"任何来源"的应用
sudo spctl --master-enable

# 按住连续输入
defaults write -g ApplePressAndHoldEnabled -bool false

# 显示隐藏文件
defaults write com.apple.Finder AppleShowAllFiles -bool true
 && killall Finder

# 菜单栏(menu bar) 时间格式调整成 3月1日 周二 18:00:00
# defaults write com.apple.menuextra.clock "DateFormat" -string "\"EEE d MMM HH:mm:ss\""

# 改变登陆背景
# 引号里边是图片路径
defaults write /Library/Preferences/com.apple.loginwindow DesktopPicture "/Users/dongquan/Pictures/桌面壁纸.jpeg"`
```
