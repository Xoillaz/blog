---
title: 笔记｜创建虚拟机环境
comments: false
categories: 工具使用
tags:
  - 最佳实践
  - incus
  - macvlan
date: 2025-07-04 19:20:08
updated: 2025-07-16 19:20:08
intro: 使用 incus 创建虚拟机环境，并配置集群与网络。
---
# 规划
我需要用多个虚拟机来模拟服务生产环境，由于计算资源有限且 GUI 没必要，选用命令行工具 incus 来做虚拟机管理，关于 incus 的介绍详见[这篇文章](https://blog.icku.eu.org/2024/02/04/Incus手册/)。

我有两台 Linux 主机，其配置及规划创建的虚拟机（默认 1c2g）列表如下：
{% codeblock %}
# debm-0: Debian 12, 4c8g
# debm-k: Debian 12 KDE, 16c16g
+--------------+----------+
|     NAME     | LOCATION |
+--------------+----------+
| 0-master     | debm-k   | # 模拟跳板机：2c4g
+--------------+----------+
| 1-nfs        | debm-0   |
+--------------+----------+
| 2-backup     | debm-k   |
+--------------+----------+
| 3-db         | debm-0   |
+--------------+----------+
| 4-web-0      | debm-k   |
+--------------+----------+
| 4-web-1      | debm-k   |
+--------------+----------+
{% endcodeblock %}

# 管理与网络
通过 `incus cluster` 可以在任意主机上管理所有虚拟机，这很方便。但是虚拟机分布在不同主机上，在默认配置下无法实现所有虚拟机之间直接通信。所以需要在 incus 里创建类型为 macvlan 的网络接口，这种接口将为虚拟机分配主机所在网络的 IP，缺点是虚拟机无法与所在主机直接通信。

{% note info %}
在实施过程中我发现 macvlan 接口在无线网卡上无法正常工作，接上网线会[方便很多](https://discuss.linuxcontainers.org/t/instance-networking-across-cluster-members/21189/12)
{% endnote %}

{% codeblock 配置incus集群（debm-k 为引导机，debm-0 为从机） lang:bash https://linuxcontainers.org/incus/docs/main/howto/cluster_form/ 官方文档 %}
xrz@debm-k ~ $ incus admin init
xrz@debm-k ~ $ incus cluster add debm-0 # 生成令牌

xrz@debm-0 ~ $ sudo incus admin init    # 复制粘贴 debm-k 生成的令牌
{% endcodeblock %}

{% codeblock 设置 macvlan 网络（配置好集群以下命令可以在任意主机上执行） lang:bash https://linuxcontainers.org/incus/docs/main/howto/cluster_config_networks/ 官方文档 %}
# 设置 vlan0 在两台主机上的网络父接口
incus network create vlan0 --target debm-k --type=macvlan parent=enp0s31f6
incus network create vlan0 --target debm-0 --type=macvlan parent=enp2s0
# 创建 vlan0
incus network create vlan0 --type=macvlan
{% endcodeblock %}

# 创建虚拟机
{% codeblock incus-batch-init.sh lang:bash %}
# 创建模板虚拟机
incus launch images:debian/12 ready --vm --config limits.cpu=1 --config limits.memory=2048MiB
sleep 3

# 安装基础软件
incus exec ready -- apt update
incus exec ready -- apt install -y tree wget curl yash lrzsz unzip ntpdate vim cron ftp rsync ufw openssh-server
incus stop ready

# 输入示例：0#1-nfs 0#3-db k#0-master k#4-web-0 k#web-1 k#2-backup
read -A instances -p "请输入[主机#虚拟机]："

for instance in "${instances[@]}"
do
        IFS="#" read host vm <<< "$instance"
        
        echo "正在复制虚拟机…"
        incus copy ready $vm --target debm-$host
        incus start $vm
done

echo "复制完毕，准备命名工作…"; sleep 3
servers=($(incus list -c ns --format csv | awk -F, '$2 == "RUNNING" { print $1 }'))
ip=200

echo '[Match]
Name=enp5s0

[Network]
Address=192.168.1.0/24
Gateway=192.168.1.1
DNS=8.8.8.8
DNS=8.8.4.4
DHCP=false' > ./sample.network

for server in "${servers[@]}"
do
        sed -Ei "/Address/s#(.*)\.(.+)/24#\1\.$ip/24#" sample.network
        incus file push ./sample.network $server/etc/systemd/network/enp5s0.network

        incus exec $server -- systemctl restart systemd-networkd
        incus config set $server snapshots.schedule @daily

        ip=$((ip+1))
done

echo "创建结果如下："
incus ls -c ns4LM
{% endcodeblock %}

# 创建结果
{% codeblock %}
+--------------+---------+------------------------+----------+---------------+
|     NAME     |  STATE  |          IPV4          | LOCATION | MEMORY USAGE% |
+--------------+---------+------------------------+----------+---------------+
| 0-master     | RUNNING | 192.168.1.201 (enp5s0) | debm-k   | 6.2%          |
+--------------+---------+------------------------+----------+---------------+
| 1-nfs        | RUNNING | 192.168.1.202 (enp5s0) | debm-0   | 13.0%         |
+--------------+---------+------------------------+----------+---------------+
| 2-backup     | RUNNING | 192.168.1.203 (enp5s0) | debm-k   | 12.7%         |
+--------------+---------+------------------------+----------+---------------+
| 3-db         | RUNNING | 192.168.1.204 (enp5s0) | debm-0   | 13.0%         |
+--------------+---------+------------------------+----------+---------------+
| 4-web-0      | RUNNING | 192.168.1.205 (enp5s0) | debm-k   | 12.9%         |
+--------------+---------+------------------------+----------+---------------+
| 4-web-1      | RUNNING | 192.168.1.206 (enp5s0) | debm-k   | 12.9%         |
+--------------+---------+------------------------+----------+---------------+
{% endcodeblock %}