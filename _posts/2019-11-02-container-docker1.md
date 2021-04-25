---
title: Docker源码剖析（一）api-server
subtitle: api server解析
layout: post
tags: [docker]
---

根据上一篇「容器概述」里面，dockcer整体的架构，我们逐一解析相关组件的核心逻辑。今天，我们要搞的是Docker的api也就是，docker server提供对外的rest api接口，其中定义了相关的访问接口（比如，容器查询、exec、inspect等操作）。其底层实现是，调用daemon组件，该组件实现了在api中定义好了的backend接口。

好了，我们来瞅瞅吧。

首先，我们来简单的回顾一下HttpServer的原理，以及HttpServer在Docker server上面是如何使用实现的。

## 一、HttpServer 



### Backend接口的定义





## 二、 Daemon中的实现



























## Image

- **distribution** 负责与docker registry交互，上传镜像以及v2 registry 有关的源数据
- **registry**负责docker registry有关的身份认证、镜像查找、镜像验证以及管理registry mirror等交互操作
- **image** 负责与镜像源数据有关的存储、查找，镜像层的索引、查找以及镜像tar包有关的导入、导出操作
- **reference**负责存储本地所有镜像的repository和tag名，并维护与镜像id之间的映射关系
- **layer**模块负责与镜像层和容器层源数据有关的增删改查，并负责将镜像层的增删改查映射到**实际存储镜像层文件的graphdriver模块**
- **graghdriver**是所有与容器镜像相关操作的执行者