---
date: 2021-07-01T11:00:00.000Z
layout: post
title: 从零实现vue3   runtime core更新篇
last_modified_at: 2024-07-01T20:28:59.000Z
categories:
  - 技术
tags: 
updated: 2025-07-06 04:07:20
---

## 开始之前

本文介绍vue3中最重要的模块——`runtime-core`的更新机制。 本节也包含vue3的`diff`算法讲解，就算你完全没接触过，并且没有算法基础，本文也将以最通俗的语言带你完全搞懂这一部分。

**注意，为了让读者顺利理解整个过程，本文将剔除一些代码，只展示核心逻辑。**

本文知识来自: [vuejs/core](https://github.com/vuejs/core)、[mini-vue](https://github.com/cuixiaorui/mini-vuehttps://github.com/cuixiaorui/mini-vue)

本文代码仓库：[view](https://github.com/exwer/view)

欢迎大家前去star！

## 回顾

在上一节`runtime-core`更新篇中，你已经知道了组件整个初始化流程，如果你还没看，可以[戳这里](https://juejin.cn/post/7119054243184508935)复习一下哦！

对于已经忘记的小伙伴，我也准备了一张流程图，帮你快速回忆起来整个流程，并且在其中标注了我们本节将要实现内容的位置：
![image-20250706003258922](https://raw.githubusercontent.com/exwer/img_bed/master/picGo/image-20250706003258922.png)

我们可以看到，更新操作的触发root一定是组件，叶子也一定是`fragment`或`text`，这一点我们在上一节也已经讲过。

而不论是初始化还是更新，vue都将这个逻辑放在了`patch`函数中，而`patch`的第一个参数即为新节点，那么接下来就很简单了，如果传入参数有这个新节点，就可以判断出是更新操作，否则，就是初始化操作。

下面，就让我们从更新的触发时机入手，直到理解整个组件更新的链路。

## 组件更新的时机

在组件注册时，我们有这样一步：`setupRenderEffect`，在更新那一章，我们介绍它是这样的：

```javascript
function setupRenderEffect(instance, vnode, container){
  // 获取proxy对象
  const { proxy } = instance
  
  // 执行render函数，并让其this指向proxy (这样this.xxx才会有值)
  const subTree = instance.render.call(proxy)
  
  // 继续遍历子树
  patch(subTree, container, instance)
}

```

可以看到，是通过它来调用`patch`，继续构建子树，那么理所应当的，如果该组件是要更新而不是初始化，也应该在这里进行判断。

该如何判断当前组件是要更新还是初始化呢





