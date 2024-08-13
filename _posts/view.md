---
date: 2024-07-01

date: 2024-07-01

layout: post
title: "view"
date: 2021-07-01T11:00:00.000Z
last_modified_at: 2024-07-01T20:28:59.000Z
categories:
  - 技术 
tags:
---

# reactivity
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