---
title: 'PXE 启动与 iPXE 界面定制 —— Matchbox 前记'
description: '基于兽耳娘深度定制的 iPXE 系统 ( •̀ ω •́ )y。'
slug: pxe-boot-and-ipxe-gui-stylization
keywords:
    - pxe
    - ipxe
    - matchbox
    - stylization
date: 2024-11-26 16:00:00+0900
lastmod: 2025-02-11 09:00:00+0900
image: cover.jpg
categories:
    - Infrastructure
    - Programming
tags:
    - PXE
    - iPXE
    - Matchbox
    - HomeLab
---

## 前言

[PXE（预启动执行环境）](https://zh.wikipedia.org/wiki/%E9%A2%84%E5%90%AF%E5%8A%A8%E6%89%A7%E8%A1%8C%E7%8E%AF%E5%A2%83)
提供了使用网络接口启动计算机的机制，也可以称为 `网络引导` 或 `网络启动`。
其核心价值在于网络化的集中管理和自动化，尤其适合需要快速、批量、标准化操作的场景。

想了解更多？请参考 [Preboot Execution Environment（archlinux wiki 英语）](https://wiki.archlinux.org/title/Preboot_Execution_Environment)

### 为什么需要 PXE？

- 不依赖本地数据存储设备（e.g. 硬盘）或本地已安装的操作系统；
- 批量部署生产系统，e.g. [kickstart（WIKIPEDIA 英语）](https://en.wikipedia.org/wiki/Kickstart_(Linux)) 无人值守安装；
- 从中央服务器远程启动安装镜像（ISO、EFI 等），无需插入 U 盘或光盘；

### iPXE 和 PXE 的区别

[iPXE](https://en.wikipedia.org/wiki/IPXE)
是预启动执行环境 (PXE) 客户端软件和引导加载程序的开源实现。
也就是说 iPXE 实际上是 PXE 下载执行的第二启动固件。
官方网站：[https://ipxe.org/](https://ipxe.org/)

> You can use iPXE to replace the existing PXE ROM on your network card, or you can chainload into iPXE to obtain the features of iPXE without the hassle of reflashing.
>
> （您可以使用 iPXE 替换网卡上现有的 PXE ROM，也可以通过链式加载（chainload）方式引导至 iPXE，既能获得 iPXE 的功能特性，又无需经历重新刷写固件的麻烦。）

通过 iPXE 可以做到 PXE 难以实现的功能，例如官网中列举的 `HTTP`、`iSCSI` 等。

在测试环境下有了 iPXE 还可以方便在多个系统中切换，
就像在网络里放了个 [Ventoy](https://www.ventoy.net/cn/index.html) 一样（

### 背景

最近心血来潮，在调研服务器网络引导相关内容，发掘可以用来支撑裸金属的自动化工作。
个人对于 IaC 还是比较感兴趣的 ~~（虽然这东西实操起来挺麻烦对于家庭实验室也没有太大的必要）~~ ，
不过会总比不会好嘛~ 同时这也是 Matchbox 的前篇笔记，搭建它的基础工作。

## 设计

启动链路：PXE -> iPXE -> 操作系统

工作流程：

1. PXE 客户端（服务器）开机后，网卡发送 PXE 请求到 DHCP 服务器；
2. DHCP 返回 IP 地址及 TFTP 服务器地址；
3. 客户端从 TFTP 下载引导文件；
4. 加载下载的引导文件；

### 服务器 BIOS 启用 PXE

服务器 BIOS 不同设置可能不同，但总体上大同小异。一般来说在 BOOT 页签选择 PXE 网卡、
`EFI Network` 等即可，必要时设置 PXE 网卡的参数。本文略。

### 设置 DHCP 下发启动信息

笔者使用新华三交换机。配置如下：

```text
#
vlan ***
(...)
dhcp server ip-pool DHCP_POOL_NAME
#
bootfile-name ipxe.efi
tftp-server ip-address TFTP_SERVER_IP
#
(...)
```

命令说明：

| 命令 | 说明 |
|------|------|
| dhcp server ip-pool | 指定 DHCP 地址池 |
| bootfile-name | DHCP 客户端使用的启动文件名 |
| tftp-server ip-address | DHCP 客户端使用的 TFTP 服务器地址 |

TFTP 服务器等需要自行搭建，本文略。

### iPXE Self Compile

iPXE 默认提供了预编译的 ISO 镜像，但是不符合笔者这里的需求，故考虑自编译。

因 `v1.21.1` 在新版本的编译器中 gcc 构建有一些小 BUG，
在这里 https://github.com/qianjunakasumi/ipxe 维护了一个分支。可供参考。

在 iPXE 中嵌入脚本：

```ipxe
#!ipxe
# Edited at 11/25/2024 qianjunakasumi. Approved

echo TACHIBANA iPXE of LYA (11/25/2024 x86_64)

:retry_dhcp
ifconf --configurator dhcp net0 || goto retry_dhcp

echo DHCP succeeded
echo Launching next level...

set matchbox-url http://matchbox.tachi:8080

chain ${matchbox-url}/assets/ipxe/level/tachibana.ipxe
```

其中 `tachibana.ipxe` 是链式启动的下一个 iPXE 脚本，定制化工作也在此实施。

### 串联

构建好 iPXE 后放入 TFTP 服务器，此时启动服务器应能正常从 iPXE 启动了。

## iPXE GUI Personalization

iPXE 脚本支持 `console` 关键字创建类似于 `GRUB` 的启动 GUI，
更多用法参考： https://ipxe.org/cmd

定制化参考：

```ipxe
#!ipxe
# Edited at 02/10/2025 qianjunakasumi. Approved

echo TACHIBANA iPXE script loaded successfully
echo Launching console...

console --picture ${matchbox-url}/assets/ipxe/console-background/fitted.png

:start
menu TACHIBANA - iPXE (02/10/2025)

item --gap                       ==== TACHIBANA IaC infrastructure
item --gap
item --default --key m matchbox  matchbox chain boot (Default)
item --gap
item --gap                       MAC: ${mac:hexhyp}, UUID: ${uuid}
item --gap                       Hostname: ${hostname}, Domain: ${domain}
item --gap                       Serial: ${serial}, Arch: ${buildarch:uristring}
item --gap
item --gap                       ==== Others
item --gap
item firpe                       FirPE
item --gap
item --gap                       ==== Advanced
item --gap
item ipxe-shell                  iPXE shell
item reload-console              Reload Console
item --key r reboot              Reboot Server
item poweroff                    Power OFF
item --key e cancelled           Exit iPXE

choose --default matchbox --timeout 5000 target && goto ${target} || goto cancelled

:matchbox
chain ${matchbox-url}/boot.ipxe

:firpe
sanboot --no-describe ${matchbox-url}/assets/firpe/FirPE-V1.9.1.iso || goto failed
goto start

:failed
echo "ERROR! Please contact customer support"
goto ipxe-shell

:ipxe-shell
echo "Type 'exit' to return console"
shell
goto reload-console

:reload-console
chain ${matchbox-url}/assets/ipxe/level/tachibana.ipxe

:reboot
reboot

:poweroff
poweroff

:cancelled
exit
```

核心关键字：

- `console --picture` 用于设置背景图片和 GUI 宽高
- `menu` 创建菜单标题栏
- `item --gap` 创建菜单不可选中的部分
- `item` 可供选择的条目

语法学习可参考： https://ipxe.org/scripting

最终效果如下：

![TACHIBANA iPXE console](ipxe-console.jpg)

嗯~ 非常满意，来自一位学弟的评价：
《基于兽耳娘深度定制的 iPXE 系统》

## Cover

[ゆめねこ🌸](https://www.pixiv.net/users/28223718)様のイラスト「[魚](https://www.pixiv.net/artworks/116178606)」です。
もしこのイラストが気に入ったら、ぜひ[「ゆめねこ🌸」様のホームページ](https://www.pixiv.net/users/28223718)もチェックしてみてくださいね！

[![魚・ゆめねこ🌸](cover.jpg)](https://www.pixiv.net/artworks/116178606)

If you believe it is being abused, please send an email to qianjunakasumi@outlook.com.
