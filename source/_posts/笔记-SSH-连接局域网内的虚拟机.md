---
title: 笔记 | SSH 连接局域网内的虚拟机
comments: true
date: 2025-03-07 22:08:34
updated: 
categories:
tags:
---

为了学习运维知识我需要 Linux 的生产环境，现有的 Windows 笔记本和 Mac Mini 主机都不能满足我的需求，我想利用 Mac 桌面来操作 Linux，在我面前的选项：
1. 在 Mac 里装虚拟机；
2. 通过 SSH 远程访问
    1. 订阅云服务器在上面跑 Linux 系统；
    2. 利用工作室现有的 Linux 主机，另外购置网卡；
    3. 在 Windows 中安装虚拟机，开放端口以供局域网访问。

考虑下来我觉得在 Windows 中安装虚拟机最有性价比。这样没有什么支出，在配置上也有一定的挑战性。另外在使用体验上，复用 Mac 桌面/软件可以提高我的幸福感，在一些比较临时的境况如果想学习工作也不会受到太大限制，拿起笔记本就能干活。以下是配置的简要过程，这里面我碰到了各种各样的问题，比如防火墙不让访问端口、Windows 可以 ping 通 Mac 但反过来不行，我推荐根据具体环境和 AI 一起去排查。不过也有 AI 解决不了的问题，比如我在 Mac 就是没法通过 ssh 连接到服务器，但在 VScode 装个插件就可以了。如果可以用工具解决就不要继续折腾了吧，这些时间值得分配在更重要的事情上。

# 背景信息
- 虚拟机连接到外部的网络类型是 NAT
- IP 信息可由 `ifconfig(Unix) \ ipconfig(Windows)` 获取
    - Server-Linux(Fedora): `192.168.132.128`
    - VMhost-Windows: `192.168.1.114`
    - Client-Mac: `192.168.1.126`

# 核心步骤
1. 在 Server 启动 openssh-server 服务
	```bash
	sudo systemctl start sshd  # 如果报告服务不存在就安装该组件
	ps -e | grep ssh           # 查看组件是否正在服务
	sudo systemctl enable sshd # 将该服务设为开机自启动
	```
	```PowerShell
	ssh root@192.168.132.128 -p 22 # 在 VMhost 验证能否从外部访问虚拟机
	```
2. 设置虚拟机的端口转发
	1. 在 VMware-编辑-虚拟网络编辑器 里选中 VMnet8（虚拟机的网卡）
	2. 更改其 NAT 设置
	3. 添加端口转发规则：将 `192.168.132.128:22` 转发到主机端口 `2222`
	```PowerShell
	ssh root@192.168.1.114 -p 2222 # 在 VMhost 检验设置是否成功
	```
3. 在 Client 通过 VisualStudioCode-RemoteSSH 访问服务