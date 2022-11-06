---
title: '[k8s]k8s部署踩坑'
tags:
  - k8s
  - deploy
categories:
  - 容器
slug: 2102019255
date: 2019-08-25 14:06:02
---
# 前言
自己搭个k8s集群，踩了一些坑

# 镜像
`kubeadm init`命令会去`k8s.gcr.io`拉镜像，这个地址是得挂代理才能上的（可以指定地址忽略代理），可以用`kubeadm config images pull`尝试一下，十有八九是不行。不想挂代理的话，用下面这个方法。

先执行`kubeadm config images list`列出镜像，输出信息中有两行`WARN`是获取版本timeout可以不理会。

接着把列出来的信息放到下面的bash脚本中，运行脚本就把镜像下载好啦(其实就是从阿里云下镜像改tag)。

```bash
images=(  # 下面的镜像应该去除"k8s.gcr.io"的前缀，版本换成上面获取到的版本
kube-apiserver:v1.15.3
kube-controller-manager:v1.15.3
kube-scheduler:v1.15.3
kube-proxy:v1.15.3
pause:3.1
etcd:3.3.10
coredns:1.3.1
)

for imageName in ${images[@]} ; do
    docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
    docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName k8s.gcr.io/$imageName
    docker rmi registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

# `token`和`ca-cert-hash`
在master进行`kubeadm init`后会输出`token`和`ca-cert-hash`，这个要记住，如果忘记了虽然可以执行`kubeadm token list`获取`token`，但是`ca-cert-hash`是不会输出的，忘记`ca-cert-hash`只能重新执行`kubeadm token create`从输出中拿到。

# ip转发
在node主机执行`kubeadm join`的时候，报

    [ERROR FileContent--proc-sys-net-ipv4-ip_forward]: /proc/sys/net/ipv4/ip_forward contents are not set to 1`
意思是没有开启ipv4转发，设置一下就好了：`echo 1 > /proc/sys/net/ipv4/ip_forward`

# 时间同步
在node主机执行`kubeadm join`的时候，一直卡住，加上`--v=2`可以输出详细信息，输出了一个信息

    I0824 21:58:46.950161   16866 token.go:146] [discovery] Failed to request cluster info, will try again: [Get https://192.168.0.113:6443/api/v1/namespaces/kube-public/configmaps/cluster-info: x509: certificate has expired or is not yet valid]`
但是我的证书没过期呀，在一个issue里一位老哥说是不是几台机器时间没同步，我在node主机上执行`date`，果然时间和master差了好久，用于是ntp命令同步了一下时间。
```bash
yum install -y ntpdate
ntpdate cn.pool.ntp.org
```

# hostname
在node主机执行`kubeadm join`的时候，要用`--node-name`指定节点名字，如果不指定，会用hostname，如果你和我一样主机是用vmware克隆出来的，几台机器的hostname都是一样的，就会执行`kubeadm join`成功，`kubectl get nodes`只有一台master(三台机hostname都是master)，`kubectl get pods --namespace kube-system`的pod也都只有一份。

# 网桥地址重复
`failed to set bridge addr: "cni0" already has an IP address different from 10.244.1.1/24`，执行`ip link delete cin0`删除**cni0**网桥。

# dashboard 404错误
输入token后显示的是404，执行`kubectl logs`发现找不到某些`Resource`，查看iusse，dashboard v1.X和新版的k8s不兼容，升到v2.x就好了。不过v2.x还没有正式版。
