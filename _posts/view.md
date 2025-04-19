---
date: 2021-07-01T11:00:00.000Z
layout: post
title: view
last_modified_at: 2024-07-01T20:28:59.000Z
categories:
  - 技术
tags: 
updated: 2025-04-16 01:16:36
---

# reactivity
## effect反向收集数据源
```javascript
export function trackEffects(dep: Set<ReactiveEffect>) {
  if (!activeEffect) return
  if (dep.has(activeEffect)) return
  
  dep.add(activeEffect)           // 正向收集：数据 -> effect
  activeEffect.deps.push(dep)     // 反向收集：effect -> 数据
}
```
反向收集的主要目的是为了清理效果。当我们需要停止一个 effect 时（比如组件卸载），我们需要：
- 找到这个 effect 被哪些数据收集了
- 从这些数据的依赖集合中删除这个 effect
没有反向收集的话，我们就需要遍历整个 `targetMap` 来找到所有包含这个 effect 的依赖集合，这样的性能开销会很大。
看看 `cleanupEffect` 函数就明白了：
```javascript
function cleanupEffect(effect: ReactiveEffect) {
  effect.deps.forEach((dep) => {
    dep.delete(effect)         // 通过反向收集的deps直接找到并清理
  })
  effect.deps.length = 0
}
```

## parent机制解决嵌套effect和无限循环问题
嵌套effect例子：
```javascript
    effect(() => {
      console.log('level 1')
      effect(() => {
        console.log('level 2')
        effect(() => {
          console.log('level 3', data.a)
        })
        console.log(data.b)
      })
    })
```
我们知道，在`track`中，响应式变量会收集`activeEffect`作为依赖，同时`activeEffect`会反向收集所有包含它的集合，以便以后的清理操作。但是在这个例子中，由于执行顺序是`level 1 -> level 2 -> level 3 -> data.b`，导致最后一次访问`data.b`时`activeEffect`实际变成了最内部的`effect`，这样就造成了`track`的乱序。
`parent`是一种类似链表的机制，来实现环境的恢复，保证`activeEffect`是准确的。
我们要保证在effect.fn()之前，它取到的activeEffect是它自己，在fn()之后，它应该把activeEffect还给他的parent。
所以我们给每个`effect`实例一个成员`parent`。借助全局变量`activeEffect`来完成连接。
```javascript
//...
parent = undefined
run(){
	//...
	let ptr = activeEffect
	while(ptr){
	}
	ptr = activeEffect
	try{
		this.parent = activeEffect //将父级设为自己
		activeEffect = this //设置当前作用域
		return this._fn() //运行fn，如果内部嵌套effect，则它拿到的parent即为目前的effect
	}
	lfinally{
		//还给父级
		activeEffect = this.parent
		this.parent = activeEffect
	}
	//...
}
//...
```

无限循环例子：
```javascript
const counter = reactive({ num: 0, num2: 0 })

//第一种
effect(() => {              
  counter.num2 = counter.num2 + 1 //counter.num2++
})

//第二种
effect(()=>{ //effect 1
	counter.num
	effect(()=>{ //effect 2
		counter.num ++
	})
})
```
在这个例子中，一个effect中同时进行了track和trigger操作，而trigger又会继续引起track，从而无限循环。
依然可以利用`parent`链条来解决这个问题，例如上述第二段代码，正常来说它的调用链应该是`effect1 -> effect2`，但是effect2会trigger到effect1，此时在`effect1`的`run`方法内，可以查找调用链上是否有自己，如果有，则说明此次是一个循环调用，应终止。
```javascript
run(){
	//...
	let parent = activeEffect
	while(parent){
		//如果在调用链上找到了自己，则说明本次是循环调用，直接终止
		if(parent === this){
			return
		}
		parent = parent.parent //一直向上找
	}
	//...上面例子的代码
}
```
当然，`第一种`无限循环的情况可以利用另外一种简便的方法处理，使用parent机制主要是应对第二种。
```javascript
triggerEffect(){
	for(const effect of deps){
		if(effect === this){
			return
		}
		effect.run()
	}
}
```
# Component
## 初始化
### ShapeFlags
使用位运算进行判断更加高效(但可读性会变差，需要权衡)

#### emit
用户使用emit时，只希望传入一个事件名,如：
```typescript
emit('add')
```

而在初始化emit时，我们需要获取组件的props内容：
```typescript
export function emit(instance, event){
	// get props in instance
}
```

这种情况下，可以使用bind:
```typescript
import {emit} from 'xxx/emit'
export function createComponentInstance(vNode){
	const component = {
		// ...
		emit: ()=>{}	
	}
	component.emit = emit.bind(null, component)
	return component
}
```

# renderer

## 双端对比算法

# 调试

- 测试组件更新功能时，组件内声明一个`isChange`响应式变量，同时`window.isChange = isChange`，这样方便在浏览器控制台触发k：`isChange.value = true`

