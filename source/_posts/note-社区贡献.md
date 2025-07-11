---
title: 笔记｜社区贡献
comments: false
date: 2025-07-11 21:09:04
updated: 2025-07-11 21:09:04
categories: 经验总结
tags:
  - 最佳实践
  - alpine
  - incus
  - git
intro: 我参与社区贡献的过程记录。
---
# Changelog
- `250711` 设置容器打包环境。
# 设置容器打包环境
```bash
incus launch images:alpine/edge/amd64 alpine-amd64
incus exec alpine-amd64 -- /bin/sh

# 根据文档进行配置：https://wiki.alpinelinux.org/wiki/Setting_up_the_build_environment_on_HDD
# 安装基本软件
apk update; apk add alpine-sdk abuild git less vim sudo 
# 按需配置abuild
vim /etc/abuild.conf
# 添加用户
adduser builder -G abuild
echo "builder ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
# 配置密钥
abuild-keygen -a; cp .abuild/*.pub /etc/apk/keys/

# 将容器的工作目录挂载到宿主机
incus file mount alpine-amd64/home/builder/aports ~/dev/aports-amd64
# 在宿主机进行git操作
cd ~/dev/aports-amd64; git clone aports.git
git config pull.rebase true
```
# TODO
- 阅读 [如何打补丁](https://wiki.alpinelinux.org/wiki/Creating_patches)、aports 里的 md
- 验证打包的工作流
	- 查阅 issue/MR（`upgrade`, `architecture`, `help wanted`）
	- 更新 APKBUILD
	- 在本地测试
	- 把 patch 提交到 aports
	- CI 自动完成真正的构建+签名+分发
- 不清楚的地方
	- 是否需要在宿主机根据架构分目录挂载
		- 依赖：`apt install qemu-user-static binfmt-support`
		- 架构：`amd64/armhf/riscv64`
	- 是否需要先 fork 再贡献
	- 怎么找出未更新的包，nucular 表现怎么样
	- 在打包之余还可以怎么贡献