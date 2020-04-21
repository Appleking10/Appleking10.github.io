---
title: vue项目开发中遇到的坑和心得（持续更新）
date: 2020-04-21
tags: [VUE,实战心得,持续更新]
---

### 前言

从学vue到工作中用到vue也有半年多，感觉很有必要来总结一下具体工作场景中遇到的一些坑已经填坑的过程。

---

#### 怎么处理数据量大的变量？

* 场景： 在应用场景中，有时候需要将一整个数据或者对象保存下来，给其它函数使用，一般来说只是展示使用。但如果放在`data()`中，其实对整个页面和浏览器不太友好。
* 原因： 众所周知，存放在`data()`中的变量，在vue的生命周期中会被实时监听而且是每一项都会被监听到，所以一个存储数据量很大且其实不需要被监听的变量就会给页面造成负担。
* 方法： 研究vue的生命周期可以知道，`data()`是在`created()`中初始化完成的，在`create()`声明一个变量，也能全局使用，但不会被实时监听到，减小性能损耗
  
```javascript
create(){
    this.arr = null;//也能全局使用
}
```

#### Vue自定义插件的构造

* 场景： 日常中我们要使用一个组件（比如弹窗组件）的方式通常是先通过Vue.component 全局或是 component 局部注册后，然后在模版中使用。但操作流程还是不够简洁方便。于是借鉴了element-ui的写法，去研究如何挂载自定义模板
* Vue.extend(options)
  * 描述： **Vue.extend**返回的是一个“扩展实例构造器”，也就是预设了部分选项的Vue的实例构造器，它常常服务于Vue.component用来生成组件，可以简单理解为当在模板中遇到该组件作为标签的自定义元素时，会自动调用“扩展实例构造器”来生产组件实例，并挂在到自定义元素上。
  * 参数类型：Object
* 用法
  * 原理：运用Vue.extend()方法，注册挂载自定义模板，并将模板显示方法挂载到Vue原型上，方便全局调用。Vue.extend 的显示与否是手动的去做组件的挂载和销毁。
  * 实战
   **1.alert.vue** //弹窗模板

```javascript
<template>
    <div id="alert" v-if="msg">
        <span class="icon">❕</span>
        <span>{{msg}}</span>
    </div>
</template>
<script>
export default {
    props: {
        msg: null //显示信息
    },
    watch: {
        msg: function(val) {
            if (val) {
                let _self = this;
                setTimeout(function() {
                    $('#alert).css({transform: "translate(-50%,0)",opacity:1})
                }, 0);
                setTimeout(function() {
                    $('#alert).css("opacity",0)
                }, 2000);
                setTimeout(function() {
                    _self.msg = null;
                    $('#alert).css("display","none").remove() //移除dom元素
                    // _self.$destroy(); //若允许生成多个实例便在用完的时候需要销毁
                }, 3000);
            }
        }
    }
};
</script>
<style scoped>
#alert{
    min-width: 200px;
    min-height: 30px;
    color: slategray;
    line-height: 30px;
    padding: 10px;
    font-size: 14px;
    position: fixed;
    top:20px;
    left:50%;
    opacity: 0;
    transform: translate(-50%,-40px);
    background: rgb(237,242,252);
    transition: all 1s ease; 
}
</style>
```

  **2.alert.js** //将创建构造器和挂载到目标元素上的逻辑抽离出来，多处可以复用
  实例化时可以向这个实例传入参数，但是需要注意的是 props 的值需要通过 propsData 属性来传递
```javascript
import Alert from './alert.vue'
import Vue from 'vue'

let Toast = {}; //定义插件对象
Toast.install = function (Vue, options) { //vue的install方法，用于定义vue插件
  let AlertConstructor = Vue.extend(Alert) //创建vue构造器
  let instance //保存vue实例
  let seed = 1; //实例id
  let index = 999; //模板的z-index
  let alertObj = {} //保存$alert对象
  instance = new AlertConstructor() //实例化vue实例
  instance.vm = instance.$mount() //拿到实例
  Object.defineProperty(Vue.prototype, "$alert", { //挂载到vue原型上，方便全局调用

    get() { //当访问该属性时，会调用get函数
      alertObj.alertMsg = options => { //调用$alert.alertMsg时
        let id = 'message_' + seed++
        instance.id = id //绑上实例id
        //实例传入参数，对应的是props里面的数据
        Object.assign(instance, options)
        index++
        document.body.appendChild(instance.vm.$el) //将实例的html模板append到html
        instance.vm.$nextTick(() => { //等dom视图刷新完毕
          instance.vm.$el.style.zIndex = index
        })
        return instance.vm.$alert
      }
      alertObj.sh = () => {
        console.log("something") //自定义方法
      }
      return alertObj //返回vue实例
    }
  })
}
export default Toast
```

  **3.main.js** //项目的全局文件引用
```javascript
import Alert from './main/alert1'
Vue.use(Alert)
```

   4.然后就可以在项目里面使用了，`this.$alert.alertMsg({msg:"something"})`