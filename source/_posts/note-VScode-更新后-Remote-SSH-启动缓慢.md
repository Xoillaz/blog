---
title: 笔记 | VScode 更新后 Remote-SSH 启动缓慢
comments: true
date: 2025-03-17 20:44:17
updated: 2025-06-29 20:02:00
categories: 经验总结
tags:
    - 远程开发
    - ssh
    - vscode
---

今天 VScode 提醒更新，我允许操作以后发现 Remote - SSH 插件总是卡在登陆后的 “正在下载 vscode server…” 提示这里不动。我看了该进程的输出，也在服务器那边反复查看下载状态，又重启了几遍，没感觉到下载进度有什么推动，我想应该又是网络代理的问题，就干脆手动下好文件传过去。

但在网上看过几篇攻略都解决不了问题，后来在这篇[较新的答案](https://zhuanlan.zhihu.com/p/426876766)受到了启发：原来其他教程提供的解压路径过时了，只要稍作修改就可以解决问题。

```bash
# 1. 在 vscode 的 “关于” 选项复制 commit_id；
# 2. 下载相应的 server 工具：https://update.code.visualstudio.com/commit:${commit_id}/server-linux-x64/stable。

# 在服务器操作，命令作用如下：
# 1. 预创建文件夹
# 2. 将 Server 工具解压到该文件夹当中
# 3. 修改相关文件，偷梁换柱
package=~/Downloads/vscode-server-linux-x64.tar.gz
commit_id=this_is_a_string_of_commit_id
target_path=~/.vscode-server/cli/servers/Stable-${commit_id}/server/

mkdir -p ${target_path}
tar zxvf ${package} --strip 1 -C ${target_path}
echo "[\"Stable-$commit_id\"]" > ~/.vscode-server/cli/servers/lru.json
```