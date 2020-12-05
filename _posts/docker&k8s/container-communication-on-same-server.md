---
layout: post
title: docker容器间通信
category: 容器以及容器编排
tags: [docker]
keywords: docker
---

1. 同主机通信
  i.默认网桥bridge上的容器只能通过ip互联：docker network inspect bridge
  ii.自定义bridge
    1).创建用户自定义bridge ：docker network create my-net
    2).将相应的服务容器加入到my-net中：docker network connect my-net xx-container
    3).通过容器名互联
    4).断开与网桥bridge的连接（dockerr0）：docker network disconnect xx-container
2. 跨主机通信（overlay）
i.暂无
