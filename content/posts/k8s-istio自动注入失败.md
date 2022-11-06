---
title: '[k8s]istio自动注入失败'
tags:
  - k8s
  - istio
categories:
  - 容器
slug: 855478903
date: 2020-02-27 00:05:20
---
查了我一天喵的
<!--more-->
# 过程
istio两种注入模式，一种是执行`istioctl kube-inject`将目标`deployment`的yaml先修改，也就是手动注入`sidecar`和`initContainer`，另一种就是在`pod`被部署的时候，利用k8s的`webhook`机制，进行自动注入。

在自动注入前，要在部署容器的`namespace`打上`istio-injection: enabled`标签，这样才会自动注入，同时，还可以指定`template`的注解:`sidecar.istio.io/inject: true`来做更小粒度的控制。

在使用`bookinfo`的demo过程中，自动注入并没有生效，甚至连`pod`都没有`create`。

执行`kubectl describe deployment productpage`查看其中一个`deployment`，发现只有一个事件， `Scaled up replica set productpage-v1-596598f447 to 1`，然后就没有然后了。

执行`kubectl describe replicaset productpage-v1-596598f447`，显示**failed calling webhook "sidecar-injector.istio.io": Post https://istio-sidecar-injector.istio-system.svc:443/inject?timeout=30s: context deadline exceeded`**，

查看`apiserver`的日志，一直提示
    
     "sidecar-injector.istio.io": Post https://istio-sidecar-injector.istio-system.svc:443/inject?timeout=30s: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)

查看`controller-manager`的日志，一直提示
    
    Event(v1.ObjectReference{Kind:"HorizontalPodAutoscaler", Namespace:"istio-system", Name:"istio-telemetry", UID:"da322eac-127a-4c78-89e6-db614d697949", APIVersion:"autoscaling/v2beta2", ResourceVersion:"10847941", FieldPath:""}): type: 'Warning' reason: 'FailedComputeMetricsReplicas' invalid metrics (1 invalid out of 1), first error is: failed to get cpu utilization: unable to get metrics for resource cpu: no metrics returned from resource metrics API

看起来是在说找不到**metrics api**？

上官网，[Istio / Sidecar Injection Problems](https://istio.io/docs/ops/common-problems/injection/)没有一个描述是符合的。github issue也没有提到要安装`metrics-server`。

最后找到一篇[博客](https://www.yp14.cn/2019/12/12/Istio%E8%87%AA%E5%8A%A8%E6%B3%A8%E5%85%A5sidecar%E5%87%BA%E9%94%99%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88/)，里面提到

![](../images/20200227005407.png)
于是乖乖安装`metrics-server`，然后注入`sidecar`的`pod`就成功创建了 = = 其实一早就看到关于**metrics api**的报错，但是我认为那是收集监控数据的，于是没鸟他，还是图样了。

# demo
![](../images/20200227012503.png)