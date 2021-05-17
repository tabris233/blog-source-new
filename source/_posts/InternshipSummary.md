---
title: InternshipSummary
date: 2017-12-31 00:00:00
author: tabris
# 图片推荐使用图床(腾讯云、七牛云、又拍云等)来做图片的路径.如:http://xxx.com/xxx.jpg
img:
# 如果top值为true,则会是首页推荐文章
top: false
# 如果要对文章设置阅读验证密码的话,就可以在设置password的值,该值必须是用SHA256加密后的密码,防止被他人识破
password: 5b3c45ef1507cff68a1fba05bda1025bc71da39135d30e0f69f4b66f919162bc
# 本文章是否开启mathjax，且需要在主题的_config.yml文件中也需要开启才行
mathjax: false
summary:
categories: 工作
tags:
  - 实习
toc: true

---



## Internship Summary

> this is the summary for my six month internship.
>
> 2018-05-24~

<!-- more -->

### 干了什么

`5月24日`入职.

半个月后开始参与**商户联调**.

`6月22号`接到**手机号同步脚本**的需求,`7月11号`上线

- 开发用时其实只有两天. 后面进行的是漫长的发布流程,公司内开发工具不熟悉.同时测试也是一个实习生,卡住了几天.

同时接到**自动化用例server**需求

`8月5号`正式方案评审结束

`8月10号`正式开发完,后一直等待测试侧的开发

`8月17号`左右接到**商户迁移对账脚本**的需求 `8月24号`开发完毕,但后期有优化

- 这个做的就太慢了, 6张表整不明白了 当然也和python语言有些关系,语言基础不够

`8月底`开始部署自动化测试用例,陆续到9月中旬正式运行.

`8月底`接到QA的**商户监控报表**需求,开始采取`监控平台报表定制`,但不能查到商户名称,无奈改成python脚本.

- 开发上线还是有问题.  分布式调度平台无法调用.

`9月10号`左右接到**深铁预测提取脚本**需求, `9月18号`会议结束,`9月25号`脚本开发完毕,等征信侧部署ditto.

> 这时候基本没有商户联调的工作了

`9月14号`接到**客服系统优化**需求,`9月底`前端功能点部分优化结束.然后开始挂起.

- 前端都不会,现学现卖,做的很慢,

> ~~`9月底`接到薪资offer,心态崩.此事不谈~~

`10月中旬`开始在做后台部分的修改方案

> 当时考虑的是2.0的兼容,但是沟通问题,导致没有理解到位

`10月中旬`接到**自动化用例的优化**小需求,工时较短,但由于依赖服务还在测试阶段 没有提发.

> 测试结束后突然又加了个优化点..

`10月中旬`接到**行业数据预拉取**需求 ,于`10月17日`方案评审 ,`10月25日正式开始编码`,`10月26日`提code review,`10月27日`提测, `11月5日`评审了代码,同日排上测试,`11月9日`测完,`11月12日`发布.

> 接到需求时 客服系统需求挂起 快结束时继续开发
>
> 并发程序开发经验匮乏.
>
> 同时出现shell脚本 '\r\n'和'\n' 的问题
>
> bug超多,

`10月中旬后期`征信侧部署结束,开始联调,后发布

`10月底,11月初`接到**自动化用例改造**需求,`11月13日`完成开发,次日联调

> cgi,server 改造,基本是从其他模块复用代码,难度不大,但cgi首次开发,进度较慢,
>
> **但是写在方案上的点竟然有遗漏,用户白名单没有配置????**
>
> 同事周5前端换人,进度延期
>
> 之前代码仓库申请的是我的git目录下 发布的时候发现不行
>
> `11月15日`申请正式代码库,被要求用新框架开发cgi,尝试改造,**半天工**后,发现框架改动较大,依赖非常不好改,遂放弃,依旧沿用老框架.

`11月15,16日`完成**客服系统**的前后端开发

> 前端的分支目录未知, 还没有把代码提交到分支上.
>
> 后面验证下就可以发布了

`11月17,20~24日`,**自动化用例**与前端联调,同时接到**广告/活动/红点查询链路优化需求**

> 前端临时换人, 导致了很多坑,本预计17号收尾的,延误一个多星期.
>
> 联调过程,虽然是开发环境缺少数据等因素耽搁了时间.同时前端工作交接出现问题,前端代码中的一处修改/一处打桩,导致两个调了很久的问题.但主要还是我的经验不足,一来导致不管是前端还是导师/leader都觉得是我的问题.......
>
> 没有对前后端参数进行仔细的对比,,,对基础工具(apache)的使用不熟练.造成时间上的严重浪费.

### 学习了什么

很多,

> 点比较杂,一些零散经验性的东西,很难列出.

- linux的使用
- 开发工具的使用
- 应该注意的问题
- 对架构有了点了解




### 现在的问题

>  能力问题还是经验问题?

经验问题是一定存在的.

- 内部工具掌握的不够
- 开发经验的不足
- 项目系统不够了解,

能力问题

- 问题定位的速度慢
- 头铁,
- 基础**不**扎实