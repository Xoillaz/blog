---
title: 笔记｜创建虚拟机环境
comments: false
categories: 工具使用
tags:
  - 最佳实践
  - incus
  - macvlan
date: 2025-07-04 19:20:08
updated: 2025-07-04 19:20:08
intro: 使用 incus 创建虚拟机环境，并配置集群与网络。
---
# 规划
我需要用多个虚拟机来模拟服务生产环境，由于计算资源有限且 GUI 没必要，选用命令行工具 incus 来做虚拟机管理，关于 incus 的介绍详见[这篇文章](https://blog.icku.eu.org/2024/02/04/Incus手册/)。

我有两台 Linux 主机，其配置及规划创建的虚拟机（默认 1c2g）列表如下：
{% codeblock %}
# debm-0: Debian 12, 4c8g
# debm-k: Debian 12 KDE, 16c16g
+------------+----------+
|    NAME    | LOCATION |
+------------+----------+
| backup-110 | debm-k   |
+------------+----------+
| master-112 | debm-k   | # 模拟跳板机：2c4g
+------------+----------+
| web-102    | debm-k   |
+------------+----------+
| web-108    | debm-k   |
+------------+----------+
| db-107     | debm-0   |
+------------+----------+
| nfs-106    | debm-0   |
+------------+----------+
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
incus exec ready -- ufw enable
incus exec ready -- ufw allow 22
incus exec ready -- ufw reload
incus stop ready

# 输入示例：0#nfs 0#db k#master k#web k#web k#backup
read -A instances -p "请输入[主机#虚拟机]："

for instance in "${instances[@]}"
do
        IFS="#" read host vm <<< "$instance"
        
        echo "正在复制虚拟机…"
        incus copy ready $vm --target debm-$host
        
        incus start $vm
        echo "等待实例启动中…"
        for _ in {1..3}
        do
                sleep 5; incus list -c ns4L
        done
        
        at=$(incus list -c n4 | sed -rn "/$vm/s#(.*)\.([0-9]+)\s(.*)#\2#gp")
        incus exec $vm -- hostnamectl set-hostname $vm-$at
        incus stop $vm; sleep 5
        incus rename $vm $vm-$at
        
        incus config set $vm-$at snapshots.schedule @daily
        echo "已完成 $vm-$at 的创建"
done

echo "重启所有实例中…"
incus start --all; sleep 5

echo "创建结果如下："
incus list -c ns4SL
{% endcodeblock %}

# 创建结果
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