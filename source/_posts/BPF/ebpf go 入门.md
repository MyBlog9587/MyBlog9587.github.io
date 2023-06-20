---
title: ebpf go 入门 翻译
date: 2023-05-15 23:05:31
categories:
- ebpf
tags: ['bpf', 'golang', 'private']
---

# 框架选择：
- 基于 Python 的 BCC 框架
- 基于 C 的 libbpf
- 基于 Go 的 Dropbox、Cilium、Aqua 和 Calico

# 选择一个 eBPF 库
在大多数情况下，eBPF 库可以帮助您实现两件事：
- 将 eBPF 程序和映射加载到内核并执行重定位，通过其文件描述符将 eBPF 程序与正确的映射相关联。
- 与 eBPF 映射交互，允许对存储在这些映射中的键/值对进行所有标准 CRUD 操作。

一些库还可以将 eBPF 程序附加到特定的钩子，尽管对于网络用例，这可以通过任何现有的 netlink API 库轻松完成


# 目标：实现一个 XDP 交叉连接应用程序
拥有一个监视配置文件并确保本地接口根据该文件中的 YAML 规范互连的应用程序

{% asset_img xdp-xconnect.png 背景图片 %}


# 





> [原文链接](https://networkop.co.uk/post/2021-03-ebpf-intro/)