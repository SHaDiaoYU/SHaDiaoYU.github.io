---
title: vue组件传参
date: 2023-03-19 19:52:02
tags:
---

#### 组件的关系可分为 父子 与 兄弟 关系

类型1：父组件向子组件共享数据

1. 父组件向子组件共享数据需要使用自定义属性。示例：

````vue
//父组件
<son :msg= 'message' :user = "userinfo"></son>
data(){
	return{
		message: 'hello',
		userinfo: {name: 'zd',age: '20'}
}
}
//子组件son内：
<template>
	<p> {{msg}}</p>
	<p> {{userinfo}}<p>
</template>
props:['msg','user'] //请注意不要修改props的数据
````

2. 子组件向父组件共享数据需要使用自定义事件。示例：

```vue
//子组件son内容
<template>
	<p>{{ count }}</p>
</template>
data() {
	return {
	count: 0
	};
}，
methods: {
	add() {
	this.count +=1
	// 修改数据时，通过 $emit()触发自定义事件
	this.$emit('numchange' , this.count)
}
}
//父组件内容 @numchange函数被触发时会调用getNewCount方法
<son @numchange= 'getNewCount'></son>
data() {
	return {
	// 定义 countFromSon 来接收子组件传递过来的数据
	coutFromSon: 0
	};
},
methods:{
	getNewCount(val) {
	this.countFromSon = val
}
}
```

类型2： 兄弟组件之间共享数据

在vue2.x中，兄弟组件之间共享数据的方案是EventBus。

EventBus的使用步骤：

1. 创建eventBus.js模块，并向外共享一个Vue的实例对象
2. 在数据*发送方* ，调用bus.$emit('事件名称'，要发送的数据)方法触发自定义事件
3. 在数据*接收方*，调用bus.$on('事件名称'，事件处理函数)方法注册一个自定义事件

示例：

```javascript
//eventBus.js模块
import Vue from 'vue'
//向外共享Vue的实例对象
export default new Vue()

//兄弟组件A（发送方）
import bus form './eventBus.js'
export default {
	data() {
		return:{
			msg: 'hello'
		}
	},
	methods: {
		sendMsg() { // $emit()触发发送事件，该事件内包含需共享的数据
			bus.$emit('share', this.msg)
		}
	}
}
//兄弟组件B（数据接收方）	
import bus form './eventBus.js'
export default {
	data() {
		return:{
			msgFromA: ''
			}
		},
	created() { //钩子函数，接收数据，执行后接受的参数被赋值给自己的data()元素中
		bus.$on('share',val=>{
			this.msgFromA = val
		}
	}


```



