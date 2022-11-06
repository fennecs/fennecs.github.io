---
title: '[网络]端口转发'
tags:
  - iptables
categories:
  - 网络
slug: 2735588161
date: 2020-05-01 23:53:39
---
之前买的GGC家的HK vps，从去年开始电信访问一直丢包，丢包率一上去，带宽再大速度也是提不上去（[[网络]TCP拥塞控制那些事](./3284953854.html)）。

于是想到买个nat机中转一下流量。

![](../images/20200502003159.png)
(2核384内存，适合用来中转流量)

![](../images/20200502000957.png)


# iptables端口转发
nat到手后，机器镜像用的是centos7，关闭firewall，把iptables装上。
```bash
# 关闭 firewalld
systemctl disable firewalld --now
# 装iptables
yum install iptables-services
# 开机启动
systemctl enable iptables.service --now
```
接着开启ipv4转发，默认iptables是关闭ipv4转发
```bash
# 临时开启 ipv4 转发
echo 1 > /proc/sys/net/ipv4/ip_forward
# 永久开启 ipv4 转发
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
# 使生效
sysctl -p
```
接着是加iptables规则，需要在nat表加入一条DNAT和一条SNAT，DNAT是修改目的地址，转发流量到HK vps；SNAT是修改源地址，保证流量回到这台机上（只有这台机知道怎么回到我家）。

假设NAT机监听`10086`端口，HK vps的ip是`104.104.104.104`，ss服务端口是`10010`，NAT机内网地址是`192.168.1.10`
```bash
# DNAT
iptables -t nat -A PREROUTING -p  tcp -m tcp --dport 10086 -j DNAT --to-destination 104.104.104.104:10010
# SNAT，--to-source ip[:port] port可以不指定，会是随机的
iptables -t nat -A POSTROUTING -p tcp -m tcp -d 104.104.104.104 --dport 10010  -j SNAT --to-source 192.168.1.10
# 应用iptables
service iptables save
# 重启
systemctl restart iptables
# 看下nat表
iptables -nL -t nat
```
此外，nat机要开放**10086**端口。

最后一步，在nat控制面板的**NAT转发策略**创建策略，创建一个映射到该机器10086的策略，就能拿到公网ip和端口了。

![](../images/20200502002852.png)

# firewall端口转发
firewall的会简单一些
```bash
# 允许防火墙伪装IP
firewall-cmd --add-masquerade --permanent
# 检查是否允许伪装IP
firewall-cmd --query-masquerade 
# 添加转发规则
firewall-cmd --permanent --zone=public --add-forward-port=port=10086:proto=tcp:toport=10010:toaddr=104.104.104.104
# 开放监听端口
firewall-cmd --zone=public --add-port=10086/tcp --permanent
```
