---
title: Redis 学习
date: 2021-04-17 21:47:23
description: ["redis学习,,,"]
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
  - redis
tags:
  - osx
---

[TOC]

> 准备好好学习下redis了
> [《Redis 设计与实现(第二版)》](https://www.kancloud.cn/kancloud/redisbook/63834)
> [redis-3.0.0 带中文注释代码](https://github1s.com/huangz1990/redis-3.0-annotated)
> [redis 最新版代码](https://github1s.com/redis/redis)
> 准备跟书看，同时对比下最新版代码，最后运行调试看下。
> 博客还不知道会不会更新。。。。

# 环境安装

我个人习惯用vscode。

C/C++ 开发环境这里不展开了，参考这个搞下就行了

简单配置下就可以断点调试了

.vscode/launch.json

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(redis-6.2.1) 启动",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceFolder}/src/redis-server",
            "args": ["${workspaceFolder}/redis.conf"],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "lldb",
            "preLaunchTask": "build",
            "setupCommands": [
                {
                    "description": "为 gdb 启用整齐打印",
                    "text": "-enable-pretty-printing",
                    "ignoreFailures": true
                }
            ]
        }
    ]
}
```

.vscode/tasks.json

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "build",
            "type": "shell",
            "command": "make",
            "args": [
                "CFLAGS=\"-g -O0\""
            ]
        }
    ]
}
```

# 阅读内容

- [ ] 数据结构
  
  - [ ] 基础数据结sdsß
    
    - [x] sds （sds.h, sds.c, sdsalloc.h)
    
    - [x] list    ()
    
    - [x] dict  (dict.h, dict.c)
    
    - [ ] zkiplist
    
    - [ ] intset
    
    - [ ] ziplist
  
  - [ ] 外部数据结构（各类对象）
    
    - [ ] 对象的类型和编码
    - [ ] 字符串对象
    - [ ] 列表对象
    - [ ] 哈希对象
    - [ ] 集合对象
    - [ ] 有序集合对象