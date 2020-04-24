---
title: VueJS学习笔记：总结
tags: Vue.js
categories: 正常的文章
date: 2020-04-24 22:47:31
---

## 写在前面

原本是打算写一个*VueJS学习笔记*专栏，用于记录vue学习过程中的一些感想，但后来想想，如此做实在是太麻烦，并且每篇文章的篇幅也会比较短，所以现在考虑直接在这一篇博文中进行总结。将学习过程中的感谢或者踩得一些坑直接记录在此一篇文章中，尽量做到每个点都短小精悍。该篇博文并非最终稿，内容会随着文章的更新不断丰富。

**官方中文文档**：

- Guide：[https://cn.vuejs.org/v2/guide/](https://cn.vuejs.org/v2/guide/)
- API：[https://cn.vuejs.org/v2/api/](https://cn.vuejs.org/v2/api/)
- Style Guide：[https://cn.vuejs.org/v2/style-guide/](https://cn.vuejs.org/v2/style-guide/)
- Examples：[https://cn.vuejs.org/v2/examples/](https://cn.vuejs.org/v2/examples/)

> 下面大部分内容都可以从官方文档中找到。这篇总结，仅将我个人认为有必要记录的进行汇总，以方便后续查找和回顾。

## 正文

1. 在选项属性或者回调方法使用箭头函数会导致在方法内使用`this`获取不到vue实例

```javascript 错误示范
created: () => console.log(this.a)
vm.$watch('a', newValue => this.myMethod())
```

2. 事件总线机制下，监听总线事件的回调方法要使用箭头函数，否则this指代总线实例。

```javascript
mounted(){ // 挂载后执行
  EventBus.$on('eventName',data => { // 这里要用箭头函数，否则this指代EventBus
	this.data = data;
  })
}
```

3. v-bind指令使用动态参数（2.6.0+）

```html
<a :[attr]="value">...</a>
<a @[event]="value">...</a>
```

`attr`和`event`会作为一个javascript表达式进行求值

4. 在DOM中使用模板时，避免使用大写字符来命名键名，因为浏览器会把attribute名全部强制转换为小写。

5. 数据一般是从父组件流向子组件的，prop的值会因父组件中数据的改变而改变。

6. 不要修改prop的值，如果一定要这么做，请使用计算属性或者data。

7. 计算属性默认只有getter，但是我们也可以提供一个setter。

```javascript
computed: {
  fullName: {
    // getter
    get: function () {
        return this.firstName + ' ' + this.lastName
    },
    // setter
    set: function (newValue) {
        var names = newValue.split(' ')
        this.firstName = names[0]
        this.lastName = names[names.length - 1]
    }
  }
}
```

8. 父组件中的data可以是一个对象也可以是一个方法，但是子组件的data属性必须是一个方法。

9. 由于JavaScript的限制，[Vue不能检测数组和对象的变化](https://cn.vuejs.org/v2/guide/reactivity.html#%E6%A3%80%E6%B5%8B%E5%8F%98%E5%8C%96%E7%9A%84%E6%B3%A8%E6%84%8F%E4%BA%8B%E9%A1%B9)。也就是说对对象的属性或数组进行修改添加删除不会反应到视图中（但值的却改变了），可以使用`vm.$set`[实例方法](https://cn.vuejs.org/v2/api/#vm-set)（全局方法的别名），或`Vue.set`[全局方法](https://cn.vuejs.org/v2/api/#Vue-set)。

```javascript
var vm = new Vue({
  data:{
    lisi:{
      age: 16    
    }
  }
})
vm.lisi.age = 18; // 直接对对象属性进行修改是不会直接响应到视图中
this.$set(this.lisi,'age',18); // 响应式
```

10. `v-if`和`v-show`的[异同](https://cn.vuejs.org/v2/guide/conditional.html#v-if-vs-v-show)：

- `v-if`指令是直接销毁和重建DOM达到让元素显示和隐藏的效果
- `v-show`指令通过修改元素的display属性让其显示或者隐藏

11. 使用`v-cloak`[指令](https://cn.vuejs.org/v2/api/#v-cloak)解决页面加载时闪烁的问题

12. [事件修饰符](https://cn.vuejs.org/v2/guide/events.html#%E4%BA%8B%E4%BB%B6%E4%BF%AE%E9%A5%B0%E7%AC%A6)、[按键修饰符](https://cn.vuejs.org/v2/guide/events.html#%E6%8C%89%E9%94%AE%E4%BF%AE%E9%A5%B0%E7%AC%A6)以及[系统修饰键](https://cn.vuejs.org/v2/guide/events.html#%E7%B3%BB%E7%BB%9F%E4%BF%AE%E9%A5%B0%E9%94%AE)

13. 使用`ref`[属性](https://cn.vuejs.org/v2/api/#ref)在父组件中[调用子组件的方法](https://cn.vuejs.org/v2/guide/components-edge-cases.html#%E8%AE%BF%E9%97%AE%E5%AD%90%E7%BB%84%E4%BB%B6%E5%AE%9E%E4%BE%8B%E6%88%96%E5%AD%90%E5%85%83%E7%B4%A0)。

```html
<div id="app">
    <button @click="add"> Click this button {{count}} times.</button>
    <children ref="child"></children>
</div>
<template id="btn">
    <div>
        <button @click="add"> Click this button {{count}} times.</button>
    </div>
</template>
<script>
    var btn = {
        template: '#btn',
        data() {
            return {
                count: 0
            }
        },
        methods: {
            add() {
                this.count++;
            }
        }
    }
    const vm = new Vue({
        el: '#app',
        data:{
            count: 0
        },
        methods: {
            add() {
                this.count++;
                this.$refs.child.add(); // 调用子组件方法
            }
        },
        components: {
            'children': btn
        }
    })
</script>
```