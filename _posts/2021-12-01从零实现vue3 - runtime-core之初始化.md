---
date: 2024-07-01

layout: post
title: "从零实现vue3   runtime core之初始化"
date: 2021-07-01T11:00:00.000Z
last_modified_at: 2024-07-01T20:28:59.000Z
categories:
  - 技术 
tags:
---

# 开始之前

本文介绍vue3中最重要的模块——`runtime-core`的第一部分实现。 之所以要分成两个部分，是因为runtime-core分为组件的初始化和更新两部分，而这两部分都比较复杂，而且相对独立。 本文将从简单的概念讲起，一步步带你理清楚vue组件从创建到渲染的全过程。

本文知识来自: [vuejs/core](https://github.com/vuejs/core)、[mini-vue](https://github.com/cuixiaorui/mini-vuehttps://github.com/cuixiaorui/mini-vue)

本文代码仓库：[view](https://github.com/love-loli/view)

欢迎大家前去star！


# 虚拟节点(vnode)

## render和h函数

vue组件渲染入口是`render`函数，运行这个函数可以使组件渲染成真实的dom。 

有人可能会觉得，我从来没写过`render`函数啊？其实不管你用哪种写法，最后都会被编译成一个`render`函数。 

例如在.vue文件（sfc）中，你可能会这样写：

```javascript
<template>
  <div>
        {{ msg }}
  </div>
</template>

<script>
export default {
  name: 'App',
    setup(){
        return {
            msg: 'hello world'
        }
    }
}
</script>
```

而大家知道，浏览器是只识别`.js`文件的，所以`.vue`文件会经过`compiler`模块的编译。

这个文件经过编译后，会变成这样：

```javascript
export default {
    name: 'App',
    setup(){
        return {
            msg: 'hello world'
        }
    },
    render(){
        return h('div', {}, this.msg)
    }
}
```

虽然在开发中我们可以直接写`render`函数（也有不少组件库是这样做的，比如`vant`）,但是为了可读性，大部分人会选择template写法。**但是请记住，template本质其实就是render函数**。

> 关于`compiler`模块的实现，请持续关注作者，本章节不做讨论

在上面代码中我们可以看到`render`函数使用了`h`函数来描述一个`dom`节点，这个`h`函数是什么呢？ 在vue中，`h`函数的作用就是创建一个虚拟节点`vnode`。 它的的实现是这样的：

```javascript
export function h(type, props, children){
    return createVnode(type, props, children)
}
```

看明白了吧，它其实就是`createVnode`函数的别名！为什么要搞个别名呢，就是为了让你写着方便嘛，因为经常出现嵌套的写法：

```javascript
h(
  'div',
    {},
    [
        h('p', {}, 'hello'),
        h('p', {}, 'hello')
    ]
)
```

如果都用`createVnode`来写，读起来岂不是很地狱？

如果你想先熟悉一下h函数，可以参考[官方文档](https://vuejs.org/guide/extras/render-function.html#creating-vnodes)

## vnode的本质

关于为什么使用vnode，网上已经有很多文章，这里就不再赘述了。 而vnode，实质上就是js的一个对象：

```javascript
const vnode = {
  type: 'div',
  props: {},
  children: 'hello world',
  el: null,
  // ...
}
```

它的作用就是描述一个dom对象。 那么我们现在思考一个问题，如何把一个vnode渲染成一个真实的dom？ 以上面的vnode为例，让我们把他渲染成一个真实dom:

```javascript
const el = document.createElement(vnode.type)
el.textContent = vnode.children
document.body.append(el)
```

完事了，虽然这只是最简单的一种情况，但是有了这个思路，我们已经成功一半了。

## 目标

现在我们知道了，用户写的每个vue组件，实际上都是一个vnode节点。 

而我们`runtime-core`的目标就是通过组件的render函数返回的vnode来生成正确的dom。 

在这个过程中，有很多东西需要处理，例如子组件、`props & emits`、事件、`slot`、生命周期等，还得提供一些便利的api供用户使用，例如`getCurrentInstance`、`provide & inject`等等。 

废话少说我们直接开始。

# 初始化主流程

## createApp

我们使用vue根组件时，是以`createApp`为入口的，在`main.js`中我们通常这样写：

```javascript
import { createApp } from 'vue'
const App = {
  name: 'App',
  setup(){
    return {}
  },
  render(){
    h('div',{},'hello world')
  }
}

const root = document.querySelector('#app')
createApp(App).mount(root)
```

上文说到，组件实质是在处理vnode，所以需要将组件变成一个vnode，在这个过程中处理所有该处理的东西，最后，用一个方法将整个组件树渲染成真实dom。 

我们把“变成vnode”这一步和“渲染”这一步分别拆成两个函数`createVnode`和`render`（注意这个render不是组件里的render） 于是:

```javascript
export function createApp(rootComponent){
  return {
    mount(rootContainer){
      // 变成vnode
      const vnode = createVnode(rootComponent)

      // 渲染成真实dom
      render(vnode, rootContainer)
    }
  }
}
```

## 组件也是vnode节点

如果你仔细看上面的代码，你可能会有一个疑惑。 按照上文所讲，`createVnode`应该是用来创建一个vnode的，有三个参数，上面不是写了吗：

```javascript
export function h( type, props, children ){
  return createVnode( type, props, children )
}
```

但是这里传的是个什么玩意？这把整个组件传进去了啊。 而整个组件应该只有`render`函数是返回一个vnode的，难道不应该是：

```javascript
const vnode = createVnode(rootComponent.render())
```

那么请你思考一个问题，我们的h函数是不是也可以这样写：

```javascript
import { Foo } from 'components/Foo'

const App = {
  // ...
  render(){
    // 注意这里
    return h('div', {}, h(Foo))
  }
}
```

看到没有，vue组件之间肯定是要可以嵌套的，而嵌套也是用的`h`函数。 虽然，我们也可以再搞个api让用户专门传组件节点，比如叫他`r`函数好了：

```javascript
render(){
  return h(
        'div',
        {},
        [
          h('p', {}, 'hello'),
          r(Foo)
        ] 
      )
}
```

但是这样写用户会觉得挺恶心，而且徒增复杂度。所以`h`函数也应该可以接受一个vue组件。 那么按照这个逻辑，一个vue组件也被视为一个`vnode`。

其实不止是组件，vnode还有很多类型。

我们先往下看，之后再回头实现`createVnode`。

## render

### vnode的树结构

来想想`render`的流程应该是怎么样的？ 上面说到，根组件也被看成了一个`vnode`，那我们就从根组件入手。

```javascript
// 这里vnode是根组件的vnode
render(vnode, rootContainer)
```

先不管组件的vnode是什么样的，我们既然要把它渲染成真实dom，就必然要拿到它的子节点，而要拿到它的子节点，就必须运行它的`render`函数吧。 比如它是这样的：

```javascript
const App = {
  // ...
  render(){
  return h(
          'div',
          {},
          [
            h('p',{},'hello'),
            h('p',{},'world')
          ]
        )
  }
}
```

按照思路，我们应该先创建一个`div`标签，然后在创建两个`p`标签，再给两个`p`标签里填上对应的内容。

仔细看看这个过程，如果我们把vnode看成一颗树，那么render的过程实际上就是在遍历这颗树。 

而每个`h`函数的第三个参数`children`，实际上就是一个子节点。例如上面这段代码的结构应该就是这样的：

![image-20220708180923890](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/acede035da8c4e649018a1e84741f0d5~tplv-k3u1fbpfcp-zoom-1.image)

**我们把每一个children都视为一个vnode，不管它是组件、html标签、还是纯文本**，这样我们在遍历的时候逻辑更加清晰。

我们给遍历的函数取名叫做`patch`，而我们的render目前直接调用patch就好：

```javascript
export function render(vnode, container){
  patch(vnode, container)
}

function patch(vnode, container){}
```

而在patch中，我们需要渲染不同类型的vnode，所以，让我们再次回到`createVnode`函数中，尝试创建不同类型的节点：

```javascript
// 由于我们可以只传一个组件，所以后面两个参数需要设置成可选
export function createVnode(type, props?, children?){
  const vnode = {
    type,
    props,
    children
  }
  // ...
  return vnode
}
```

首先三个基本参数`type`、`props`、`children`是需要的，`type`就表示html标签的类型，**但是，如果我们传入的是一整个组件的话，此处type就是整个组件**，这里需要注意一下。

### vnode的几种类型

为了区分不同的节点，我们需要给vnode添加一个属性，就叫`shapeFlag`吧：

```javascript
const shapeFlag = {
  ELEMENT: 'element',
  TEXT_CHILDREN: 'text_children',
  ARRAY_CHILDREN: 'array_children',
  COMPONENT: 'component',
}
```

目前，我们遇到了两大类型的`vnode`，分别是元素节点和组件节点。

而元素节点里又可以分为：数组元素节点、纯文本元素节点。

结合实例更容易理解：

```javascript
// 首先是个元素节点， 又因为children是数组，所以也是个数组元素节点
h(
  'div',
  {},
  [
    // 组件节点
    h(Foo),
    
    // 元素节点，同时children为纯文本，所以是纯文本元素节点
    h('p', {}, 'text_children'),
    
    //元素节点
    h('p', {}, h('span', {}, 'text_children'))
  ]
)
```

我们在`createVnode`中，通过判断children的类型，得到vnode的类型，然后赋值给`shapeFlag`属性：

```javascript
export function createVnode(type, props?, children?){
  const vnode = {
    type,
    props,
    children,
    shapeFlag: []
  }
  // 首先通过type区分组件节点和元素节点
  // 如果type是object，则是组件节点
  if(isObject(type)){
    vnode.shapeFlag = 'COMPONENT'
  }
  
  // 如果不是组件节点，则根据children进行区分
  // children是字符串，则是纯文本节点
  else if(isString(children)){
    vnode.shapeFlag.push('TEXT_CHILDREN')
  }
  // children是数组，则是数组节点
  else if(isArray(children)){
    vnode.shapeFlag.push('ARRAY_CHILDREN')
  }
  // 最后把普通节点类型添加进去
  else{
    vnode.shapeFlag.push('ELEMENT')
  }
  
  return vnode
}
```

## patch

接下来，在`patch`中，我们就可以对不同类型的节点做不同的操作了，由于对应操作比较复杂，我们这里拆分成不同的函数：

```javascript
function patch(vnode, container){
  const { shapeFlag } = vnode
  
  if(shapeFlag === 'COMPONENT'){
    mountComponent(vnode, container)
  }
  else {
    mountElement(vnode, container)
  }
}
```

在这两个`mount`函数里，我们分别进行渲染。

根据节点类型判断规则，我们应该可以想到，一颗vdom树的根节点一定是一个组件节点，叶子节点一定是纯文本节点。

所以，只要一个vnode有children，我们就要不停的调用patch，直到遇到纯文本节点（叶子节点），这样才能保证整颗树被渲染完毕。

整个vdom树的遍历过程如图所示：

![image-20220708192822780](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2d078ed4db8649108db0aedd289d3e2e~tplv-k3u1fbpfcp-zoom-1.image)

接下来，我们逐个实现，先从`processComponent`入手。

### 初始化组件节点

首先，为了渲染的完整进行，我们最后一定会去渲染组件节点的`children`，而组件节点只有一个`type`属性，我们怎么获得它的children呢？

上文已经提到，那就是执行它的`render`函数：

```javascript
function processComponent(vnode, container){
  const component = vnode.type //注意！vnode.type才是组件对象
  // ...
  const subTree = component.render() //这里subTree即是render函数返回的vnode节点
  
  //最后继续遍历subTree
  patch(subTree, container)
}
```

我们把这一步抽成一个函数叫做`mountComponent`：

```javascript
function processComponent(vnode, container){
  mountComponent(vnode, container)
}
function mountComponent(vnode, container){
  const component = vnode.type
  const subTree = component.render()
  patch(subTree, container)
}
```

但是component的处理并不是这么简单的，现在我们回顾一下vue组件的写法:

```javascript
export default {
  setup(){
    return {
      msg: 'hello'
    }
  },
  render(){
    // 注意这个this.msg
    return h('div', {}, this.msg)
  }
}
```

可以看到在render函数里可以直接使用`this.xxx`访问`setup`函数返回的数据（data、method等同理），学过JS的都知道这是不可能的，但是既然想这样方便用户使用，我们自然就得处理一下。

#### component处理流程

像这种需求，我们通常会想到使用`call`之类的命令来解决。但是要知道一个组件上可不只有`setup`这一个属性，如果这样写到后期加功能的时候就会很麻烦。

所以这里我们需要创建一个`proxy`，统一管理component的`get`和`set`。

而在处理proxy之前，我们需要把component的处理流程打通，我们先拆分一下`mountComponent`函数：

```javascript
function processComponent(vnode, container){
  mountComponent(vnode, container)
}
function mountComponent(vnode, container){
  const instance = createComponentInstance(vnode)
  setupComponent(instance)
  setupRenderEffect(instance, vnode, container)
}
```

我们一步步来讲解每个函数的作用，同时进行实现。

#### createComponentInstance

我们都知道，`vnode.type`里存储着的是component对象，但是这个对象是用户的配置，作为开发者，我们需要处理很多情况，可能需要往对象上挂载很多其他属性，比如：

```javascript
const component = {
  setupState, // setup函数返回的对象或者函数
  props: {},
  emit: ()=>{},
  slots: {},
  provides: {},
  parent,
  // ...
}
```

可以看到，我们需要处理的属性很多，而把它们挂载在一个对象上是十分方便的。所以我们可以把自己的`component`看作用户创建的component的实例（instance)，而根据component创建instance的方法就是`createComponentInstance`：

```javascript
export function createComponentInstance(vnode){
  const instance = {
    vnode,
    component: vnode.type, //这里是为了更方便的取到component
    setupState: {}
  }
  return instance
}
```

#### setupStatefulComponent

我们创建了instance之后，还需要在instance上做进一步的处理，例如初始化props、slots等等。上面提到的component proxy也是在这一步完成。

为什么不在createComponentInstance里全部处理完呢？这样写是为了职责分离，createComponentInstance只负责初始化，进一步处理则放在setupComponent中:

```javascript
export function setupComponent(instance){
  // initProps(instance)
  // initSlots(instance)
  // ... 
  setupStatefulComponent(instance) // 我们把有状态的组件单独用此函数处理
}
```

目前我们的需求仅针对有状态的组件，所以取名叫`setupStatefulComponent`。

接下来我们就在这个函数里实现我们的proxy。

##### 实现component proxy

```javascript
function setupStatefulComponent(instance){
  // 首先拿到用户创建的component
  const Component = instance.type
  
  const { setup, render } = Component
  if(setup){
    // 执行setup函数，得到返回值
    const setupResult = setup()
    
    // setup也可以返回render函数，这里只考虑返回对象的情况
    if(isObject(setupResult)){
      //把返回值作为属性挂载到instance上面
      instance.setupState = setupResult
    }
  }
  
  if(render){
    // 把render函数也挂载到instance上面
    instance.render = render
  }
  
  // 创建代理对象
  instance.proxy = new Proxy(instance, PublicInstanceProxyHandlers)
}
```

我们把proxy的handler抽离成一个文件componentPublicInstance.ts:

```javascript
export const PublicInstanceProxyHandlers = {
  get(target, key){
    const { setupState } = target
    if(key in setupState){
      // 如果要取的值在setupState里 则直接返回,实现了this.xxx的功能
      return setupState[key]
    }
  }
}
```

#### setupRenderEffect

在流程之初我们就说到，组件vnode的最后一步一定是继续`patch`组件的`subTree`，这就是`setupRenderEffect`做的事情了：

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

这样以来，我们组件vnode的整个初始化流程就已经结束了。

#### 其他功能

你可能会注意到，像`props`、`emit`等等很多功能还没有实现，其实，我们的架子已经搭好，再加功能也就是水到渠成的事情，基本没有什么难点了。

当然，本项目已经实现了component大部分功能，但如果在一篇文章全部讲清楚是不可能的，所以欢迎有兴趣的可以直接看[我的代码]("https://github.com/love-loli/view")。（这里面我觉得实现起来比较复杂的主要是provide/inject，如果不懂欢迎提出疑问）

### 初始化element节点

处理element节点的方法就和组件节点完全不同了，我们先把视角拉回入口：

```javascript
processElement(vnode, container)
```

而元素vnode的结构是和`createVnode`对应的：

```javascript
h('div', {}, 'hello')

//就是h函数
createVnode(type, props, children)
```

我们以一个相对完整的element vnode为例：

```javascript
h(
  'div',
  {
    id: 'home',
    class: ['red', 'hard'],
    onCLick(){}
  },
  [
    h('div', {}, this.msg),
    h(Foo)
  ]
)
```

这个函数转化成vnode后：

```javascript
{
  type: 'div',
  props: {
    id: 'home',
    class: ['red', 'hard'],
    onClick(){}
  },
  children: [
    h('div', {}, this.msg),
    h(Foo)
  ]
}
```

所以，我们只要在流程中分别处理`type`、`props`和`chidren`即可。

#### 根据type创建节点

我们在处理element vnode时，最后一步一定是渲染成真实dom。

而type即为html的标签名，所以我们可以先把这两步写出来：

```javascript
function processElement(vnode, container){
  const el = document.createElement(vnode.type)
  
  // ...
  
  container.append(el)
}
```

#### 处理props

首先，需要遍历所有的props：

```javascript
if(vnode.props){
  for( key in vnode.props ){
    // ...
  }
}
```

通常，props分为两种，一种是挂载在标签上的普通属性，例如`id`、`class`等等，这种我们可以用`setAttribute` API来处理：

```javascript
if(vnode.props){
  for( key in vnode.props ){
    el.setAttribute(key, props[key])
  }
}
```

另一种是事件，例如`onClick`、`onChange`等等，我们首先要判断出它是事件prop，然后可以用`addEventListener`注册上事件。

判断的方法，其实也很简单，在vue3中所有事件都以`on`开头，而且是`驼峰式`命名。

大家可能用`@click`这种形式比较多，其实这个`@`只是vue提供的一个语法糖，该事件本质上就是`onClick`。

而我们在注册事件时，则需要把`onClick`这种名称转换为`click`这种原生事件名。

```javascript
if(vnode.props){
  for( key in vnode.props ){
    // 开头是on 后面第一个字母为大写（驼峰式），则为事件属性
    if(/^on[A-Z]/.text(key)){
      //转换成有效事件名
      const eventName = key.slice(2).toLocaleLowerCase()
      //注册事件
      el.addEventListener(eventName, props[key])
    }
    else{
      el.setAttribute(key, props[key])
    }
  }
}
```

#### 处理children

通常，children有两种情况，一种是字符串，也就是`textVnode`，我们的叶子节点。另一种是数组，也就是还需要往下继续遍历。

对于叶子节点，我们直接使用`textContent`将其添加到元素上即可。

而对于数组，我们需要遍历它，并对每个成员再次进行`patch`。

```javascript
if(vnode.children){
  // 别忘了shapeFlag可以直接判断children的类型
  if(vnode.shapeFlag === ShapeFlags.TEXT_CHILDREN){
    // 直接渲染纯文本
    el.textContent = vnode.children
  }
  else if(vnode.shapeFlag === ShapeFlags.ARRAY_CHILDREN){
    // 继续遍历
    for (const child of vnode.children){
      //注意第二个参数需要传我们创建的节点，而不是container
      patch(child, el)
    }
  }
}
```

这样，element节点的处理就完成了，完整函数：

```javascript
function mountElement(vNode, container: Container) {
  const el = (vNode.el = document.createElement(vNode.type))
  const { children, props, shapeFlag } = vNode

  if (props) {
    for (const [key, val] of Object.entries(vNode.props)) {
      if (/^on[A-Z]/.test(key)) {
        const eventName = key.slice(2).toLocaleLowerCase()
        el.addEventListener(eventName, val)
      }
      else {
        el.setAttribute(key, val)
      }
    }
  }

  if (children) {
    if (shapeFlag & ShapeFlags.TEXT_CHILDREN)
      el.textContent = vNode.children
    else if (shapeFlag & ShapeFlags.ARRAY_CHILDREN)
      mountChildren(vNode, el)
  }

  container.append(el)
}
```

#### 其他类型的element vnode

除了以上介绍的element主流程，其实还存在一些特殊的element，比如：

无标签纯文本`text`节点：

```javascript
h('hello, world')
```

`Slot`节点:

```javascript
h(Foo, {}, {
  header: ()=>h('p',{}, 'header'),
  footer: ()=>h('p',{}, 'footer')
}
```

以及和slot配套的`fragment`节点。

其实实现起来不难，有兴趣的可以自己挑战。遇到问题来看看我的写法：[代码仓库](https://github.com/love-loli/view)

# 下期预告

下篇文章将会带大家一起分析组件、element的更新过程，其中涉及到面试高频的`diff`算法，也算是喜闻乐见了吧，喜欢本文的话可以继续关注～
