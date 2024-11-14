---
date: 2021-07-01T11:00:00.000Z
layout: post
title: view
last_modified_at: 2024-07-01T20:28:59.000Z
categories:
  - 技术
tags: 
updated: 2024-11-14 23:45:39
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
没有反向收集的话，我们就需要遍历整个 targetMap 来找到所有包含这个 effect 的依赖集合，这样的性能开销会很大。
看看 cleanupEffect 函数就明白了：
```javascript
function cleanupEffect(effect: ReactiveEffect) {
  effect.deps.forEach((dep) => {
    dep.delete(effect)         // 通过反向收集的deps直接找到并清理
  })
  effect.deps.length = 0
}
```

## parent机制解决effect嵌套
例子：
```javascript
const counter = reactive({ num: 0, num2: 0 })

effect(() => {              // effect1
  console.log('outer')
  effect(() => {           // effect2
    console.log('inner', counter.num)
  })
  counter.num2++
})
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