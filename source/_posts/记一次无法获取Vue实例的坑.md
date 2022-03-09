---
title: 记一次无法获取Vue实例的坑
tags: Vue.js
categories: 正常的文章
date: 2020-04-22 22:16:44
---

> 不要在选项属性或回调上使用箭头函数，比如`created: () => console.log(this.a)`或`vm.$watch('a', newValue => this.myMethod())`。因为箭头函数并没有`this`，`this`会作为变量一直向上级词法作用域查找，直至找到为止，经常导致`Uncaught TypeError: Cannot read property of undefined`或`Uncaught TypeError: this.myMethod is not a function`之类的错误。

<!-- more -->

错误示范：

```html
<el-button type="primay" icon="el-icon-plus" @click="add"></el-button>
<p>{{count}}</p>
<script>
    const app = new Vue({
        // ...
        data: {
            count: 1
        },
        methods: {
            add: () => this.count++
        }
    });
</script>
```

正确示范：
```javascript
<script>
    const app = new Vue({
        // ...
        methods: {
            add: function () {
                this.count++
            },
            minus() {
                this.count--
            }
        }
    });
</script>
```