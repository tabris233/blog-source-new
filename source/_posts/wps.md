---
title: "MacOS 安装 WPS 国际版及汉化[不需要汉化了]"
date: 2022-11-11 23:00:35
description: "安装并汉化 WPS 国际版"
categories:
  - 工具
tags:
  - WPS
---

# 背景

偶尔需要处理 Excel、Word、PPT 的时候还是需要使用 Office 软件，但 MS Office 需要收费，又不想用破解版，于是选择了 WPS ，但 WPS 国内版的乱七八遭东西实在太多。最近发现 WPS 居然有国际版，国际版的就良心多了，没啥乱七八遭的东西。



---



**下面的不用看了，现在国际版已经默认带中文了，从国际版官网下载就好了。 -> [https://www.wps.com/](https://www.wps.com/)**

**下面的不用看了，现在国际版已经默认带中文了，从国际版官网下载就好了。 -> [https://www.wps.com/](https://www.wps.com/)**

**下面的不用看了，现在国际版已经默认带中文了，从国际版官网下载就好了。 -> [https://www.wps.com/](https://www.wps.com/)**


# 安装

安装国际版首先得能够「出国留学」，然后执行 `brew` 命令就好了。

```shell
# 先安装国内版拿到语言包之后再安装国际版
brew install wpsoffice
# 注意 wpsoffice-cn 这个是国内版。
```

# 汉化

因为是提供给国际友人的，国际版没有自带中文。所以需要自己手动汉化一下才行。

汉化首先得有 WPS 中文的语言包，最简单的还是从国内版 WPS 中拿到。

1. 先安装一遍国内版，然后把国内版的软件移到其他地方备份好。

  ```bash
  brew install wpsoffice-cn
  ```

  APP 安装的目录在 `/Applications/wpsoffice.app/` 在终端执行 `cp -r` 复制到另一个目录就好了就好


2. 删除国内版，安装国际版

   > 略

3. 找到中文语言包。

  语言包在这个目录下

  ```bash
  /Applications/wpsoffice.app/Contents/Resources/office6/mui
  ```

  从国内版复制 `zh_CN`目录到国际版的相同目录下

4. 设置中文

   打开 WPS 国际版，【设置】-【Switch Language】，选择文再重启 WPS 就是中文了。部分地方还没有中文翻译不过不影响使用。


