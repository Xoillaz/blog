---
title: 笔记｜批量设置SSH
comments: false
date: 2025-07-09 20:38:21
updated: 2025-07-09 20:38:21
categories: 工具使用
tags:
  - 最佳实践
  - incus
  - ssh
  - sudo
intro:
---
# 前言
经【创建虚拟机环境】，现拥有以下实例可在对应的宿主机通过 `incus exec [vm] -- bash` 进行访问，但为了学习生产流程，须在每台服务器配置 SSH 访问。

{% codeblock %}
+------------+---------+------------------------+-----------+----------+
|    NAME    |  STATE  |          IPV4          | SNAPSHOTS | LOCATION |
+------------+---------+------------------------+-----------+----------+
| backup-110 | RUNNING | 192.168.1.110 (enp5s0) | 0         | debm-k   |
+------------+---------+------------------------+-----------+----------+
| db-107     | RUNNING | 192.168.1.107 (enp5s0) | 0         | debm-0   |
+------------+---------+------------------------+-----------+----------+
| master-112 | RUNNING | 192.168.1.112 (enp5s0) | 0         | debm-k   |
+------------+---------+------------------------+-----------+----------+
| nfs-106    | RUNNING | 192.168.1.106 (enp5s0) | 0         | debm-0   |
+------------+---------+------------------------+-----------+----------+
| ready      | STOPPED |                        | 0         | debm-0   |
+------------+---------+------------------------+-----------+----------+
| web-102    | RUNNING | 192.168.1.102 (enp5s0) | 0         | debm-k   |
+------------+---------+------------------------+-----------+----------+
| web-108    | RUNNING | 192.168.1.108 (enp5s0) | 0         | debm-k   |
+------------+---------+------------------------+-----------+----------+
{% endcodeblock %}

# 脚本
{% tabs SSH配置 %}
<!-- tab SSH 访问过程 -->
- 外部登录到管理机 `master-112` 
	- 相关配置：修改管理机的 SSH 端口、仅允许免密登陆
- 再由 `master-112` 以 `root` 用户登录到被管理的服务器上
	- 相关配置：服务器仅允许免密登录，仅允许来自同一子网的 SSH 请求
<!-- endtab -->
<!-- tab 我的配置思路 -->
- 于 `debm-0` 执行 `useradd-ops.sh`
	- 出于安全考虑，在所有的虚拟机上新增用户 `opsuser` 用于执行 SSH 远程操作
	- 修改每个虚拟机的防火墙规则，仅允许相同网段的入站 SSH 流量
	- 允许 `opsuser` 执行 `sudo` 不需要输入密码，开启日志功能
- 于 `master-112` 执行 `ssh-auth.sh`
	- **依赖：须预先安装 sshpass**
	- 向所有服务器分发公钥
	- 修改 `sshd` 配置（请注意 `ssh_config` 影响向外登录，`sshd_config` 影响向内登录）
	- 检查配置修改情况
<!-- endtab -->
{% endtabs %}

{% codeblock useradd-ops.sh lang:bash %}
servers=($(incus list -c ns --format csv | awk -F, '$2 == "RUNNING" { print $1 }'))

for server in "${servers[@]}"
do
  echo "正在配置服务器 $server …"

  incus exec $server -- bash -c '
	useradd -m opsuser
    echo "opsuser:07090709" | chpasswd
    usermod -aG sudo opsuser
    chsh -s /usr/bin/yash opsuser
    
    ufw default deny incoming
    ufw default allow outgoing
    ufw allow from 192.168.1.112/24 to any port 22 proto tcp
    ufw --force enable
    
    visudo
    # 在打开的文件结尾添加如下内容：
    # opsuser ALL=(ALL) NOPASSWD: ALL
    # Defaults        logfile=/var/log/sudo.log
  '
done
{% endcodeblock %}

{% codeblock ssh-auth.sh lang:bash %}
ip_list=(102 106 107 108 110)

# 定义函数以便重复调用
ops() {
  local ip="$1"
  shift  # 移除第一个参数(IP)，剩余部分作为命令
  ssh -T "opsuser@192.168.1.${ip}" "$@"
}

echo "创建公私钥…"
if [ ! -f /home/opsuser/.ssh/id_rsa ]; then
  echo "生成新密钥对..."
  ssh-keygen -f /home/opsuser/.ssh/id_rsa -N '' >> /tmp/ssh_auth.log 2>&1
fi

echo "分发公钥…"
for ip in ${ip_list[@]}; do
  sshpass -p '07090709' ssh-copy-id -o StrictHostKeyChecking=no opsuser@192.168.1.$ip >> /tmp/ssh_auth.log 2>&1

  echo "验证免密登录：主机名为 $(ops $ip hostname)"

  echo "修改sshd配置…"
  ops $ip 'sudo sed -ri "s/^#(Password.*)\s.*/\1 no/" /etc/ssh/sshd_config'
  ops $ip 'sudo sed -ri "s/^#(Pubkey.*)\s.*/\1 yes/" /etc/ssh/sshd_config'
  
  echo "重启sshd服务…"
  ops $ip sudo systemctl restart sshd
done

echo "修改管理机sshd配置…"
sudo sed -ri "
  s/^#(Port)\s.*/\1 2233/; 
  s/^#(Password.*)\s.*/\1 no/; 
  s/^#(Pubkey.*)\s.*/\1 yes/
" /etc/ssh/sshd_config

echo "重启管理机sshd服务…"
sudo systemctl restart sshd

echo "检查服务器配置…"
for ip in ${ip_list[@]}; do
  ops $ip hostname
  ops $ip "sudo grep -E '(opsuser|logfile)' /etc/sudoers"
  ops $ip "sudo grep -E '^(Password|Pubkey)' /etc/ssh/sshd_config"
done

echo "检查管理机配置…"
grep -E '^(Password|Port|Pubkey)' /etc/ssh/sshd_config
{% endcodeblock %}