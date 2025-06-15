---
title: hello,world
date: 2025-06-12 10:31:33
tags: react
categories: react,useEffect
---

在 react 中如何管理服务端状态？相信很多小伙伴的第一反应就是： 在 useEffect 里发送请求，然后结合 useState 来管理状态。

下面我们来看下，使用这种方式存在什么问题，以及如何解决这些问题，最后我们会看看其他更好的方式。

# 在useEffect里发送请求，然后结合useState来管理状态
通常我们会这样写：

