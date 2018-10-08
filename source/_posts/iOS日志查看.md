---
title: iOS日志查看
date: 2017-07-30 17:29:00
tags:
---

查看 iOS 设备System Log方法

1. syslogd to /var/log/syslog (Cydia安装，已不适用)
2. idevicesyslog (github开源，有些问题)
3. Xcode -> Deveices -> Deice -> 左下角三角形按钮
4. socat (越狱设备上查看)

<!-- more -->

本文主要针对后两个做介绍。

## socat介绍
### 安装
iPhone上打开Cydia，搜索``SOcket CAT``安装。

### 命令行开启
通过``ssh``登陆到手机 openSSH，然后终端输入命令

```
iPhone:~ root# socat - UNIX-CONNECT:/var/run/lockdown/syslog.sock
========================
ASL is here to serve you
>

```

### 使用

- run ``help``

```
iPhone:~ root# socat - UNIX-CONNECT:/var/run/lockdown/syslog.sock
========================
ASL is here to serve you
> help
Commands
    quit                 exit session
    select [val]         get [set] current database
                         val must be "file" or "mem"
    file [on/off]        enable / disable file store
    memory [on/off]      enable / disable memory store
    stats                database statistics
    flush                flush database
    dbsize [val]         get [set] database size (# of records)
    watch                print new messages as they arrive
    stop                 stop watching for new messages
    raw                  use raw format for printing messages
    std                  use standard format for printing messages
    *                    show all log messages
    * key val            equality search for messages (single key/value pair)
    * op key val         search for matching messages (single key/value pair)
    * [op key val] ...   search for matching messages (multiple key/value pairs)
                         operators:  =  <  >  ! (not equal)  T (key exists)  R (regex)
                         modifiers (must follow operator):
                                 C=casefold  N=numeric  S=substring  A=prefix  Z=suffix

```

- run ``watch``

查看全部 iOS 设备日志

- 根据PID筛选日志

```
iPhone:~ root# ps aux | grep WeChat
mobile    1230   0.4  6.6   904112  67892   ??  Ss    3:48PM   0:04.05 /var/containers/Bundle/Application/DBF1DD69-5E16-4B20-849F-2774CEF984A6/WeChat.app/WeChat
root      3141   0.0  0.0   525920     16 s001  R+    3:51PM   0:00.00 grep WeChat

```

mobile的**PID 1230**

拿到微信PID后，查看微信日志

```
iPhone:~ root# socat - UNIX-CONNECT:/var/run/lockdown/syslog.sock
	========================
	ASL is here to serve you
	> * PID 1230
	Feb 14 15:03:09 iPhone WeChat[1230] <Warning>:  INFO: Reveal Server started (Protocol Version 25).
Feb 14 15:03:46 iPhone WeChat[1230] <Warning>:  INFO: Reveal Server stopped.
Feb 14 15:03:46 iPhone WeChat[1230] <Warning>: /BuildRoot/Library/Caches/com.apple.xbs/Sources/ExternalAccessory/ExternalAccessory-329.40.4/EAAccessoryManager.m:__51-[EAAccessoryManager _checkForConnectedAccessories]_block_invoke-632 ending background task
Feb 14 15:04:15 iPhone WeChat[1230] <Warning>:  INFO: Reveal Server started (Protocol Version 25).
Feb 14 15:05:28 iPhone WeChat[1230] <Warning>:  INFO: Reveal Server stopped.
Feb 14 15:05:28 iPhone WeChat[1230] <Warning>: /BuildRoot/Library/Caches/com.apple.xbs/Sources/ExternalAccessory/ExternalAccessory-329.40.4/EAAccessoryManager.m:__51-[EAAccessoryManager _checkForConnectedAccessories]_block_invoke-632 ending background task
Feb 14 15:05:59 iPhone WeChat[1230] <Warning>:  INFO: Reveal Server started (Protocol Version 25).
Feb 14 15:06:02 iPhone WeChat[1230] <Warning>:  INFO: Reveal Server stopped.
Feb 14 15:06:02 iPhone WeChat[1230] <Warning>: /BuildRoot/Library/Caches/com.apple.xbs/Sources/ExternalAccessory/ExternalAccessory-329.40.4/EAAccessoryManager.m:__51-[EAAccessoryManager _checkForConnectedAccessories]_block_invoke-632 ending background task

```

这样，我们就可以调试我们自己写的Tweak了，看它的日志了。

- 离开socat

run ``quit``

### Xcode 查看

Xcode -> Devices -> iOS 设备

![watchXcodeLog](../../../../images/watchXcodeLog.png)