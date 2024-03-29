---
title: Vue插件--超长单和分级列表渲染优化
date: 2021-03-26
tags: [Vue实战,技能篇]
---

### 前言

组件仓库链接： [组件仓库](https://github.com/Appleking10/virtual-list)
+ 组件效果图：
![组件效果图](/img/3.png)

+ 组件应用:

<img src="/img/4.png" alt="呈现效果" width="40%">
<img src="/img/5.png" alt="呈现效果" width="40%">



因为工作需要，需要处理一个10w+数据量的transfer，所以将公司原先组件进行重构和升级。
这篇教程主要内容为：
1. 前提基础学习：GUI渲染时机、常见的超长列表渲染
2. 超长单列表的渲染以及优化
3. 再1的基础上，兼容分级列表
4. 选择器transfer的构造
5. 封装成插件，发布到npm上(to-do-list)

### 1. 前提基础学习

#### a. GUI渲染时机

首先了解这个，要先了解js引擎是怎么工作的。

js引擎是单核引擎，也就是说，js引擎只能开一条线程来处理进程。我们可以把这条线程，看成**执行栈**，里面会顺序推入要执行的代码块（比如函数），执行这块代码再去处理下块代码。

那遇到异步任务怎么实现异步呢，关于这个可以参考下面流程图:

<img src="/img/1.png" alt="流程图" width="100%">

①首先可以将一个js脚本看成一个宏任务，然后从上至下依次执行代码，遇到异步任务。②将异步任务分类，如果是微任务（如`Promsie`）就将微任务的回调函数放入微任务队列，如果是宏任务，就等宏任务成功或者时间到了（此时js引擎会将这个任务挂起，继续执行当前的同步代码），将成功回调放入宏任务队列。③当前同步代码全部执行完毕后，js引擎会去触发`EventLoop轮询处理线程`，先去看当前微任务队列是有有任务，若有任务，从队头任务开始执行，④等当前微任务队列清空后，**浏览器会进行一次GUI渲染**，⑤然后会去取一个宏任务放到执行栈，开始执行同步代码，遇到异步任务重复②

以上便是EventLoop的原理。得出结论：**GUI渲染是在当前任务清空之后，执行下一个宏任务之前，进行的！**

* 举例说明：要如何去测试函数和页面渲染性能呢
  
```html
<div id="container"></div>
```

```javascript
    let total = 100000;
    let timer = Date.now()
    for(let i = 0;i < total; i++>){
        let li = document.createElement('li')
        li.innerHtml = i;
        document.getElementById('container').appendChild(li)
    }
    console.log(Date.now() - timer) //计算执行时间

```
这段函数的本意是要计算total个li渲染到页面花了多少时间，但运行后会发现，输出的时间很短，但页面还未渲染完成100000个li。原因就是因为js引擎会将当前所有同步代码执行后，才会进行GUI渲染。对于js引擎来说，遍历100000次很轻松，但渲染100000个li却没有那么友好，因此会有一段白屏时间。

那我们如何去计算函数从执行到页面渲染完毕的时间呢？可以在输出语句加上个定时器（宏任务），在执行定时器的之前，GUI渲染已经完成

```javascript
    let total = 100000;
    let timer = Date.now()
    for(let i = 0;i < total; i++>){
        let li = document.createElement('li')
        li.innerHtml = i;
        document.getElementById('container').appendChild(li)
    }
    setTimeout(()=>{
        console.log(Date.now() - timer)
    },0) //计算执行+渲染时间
```
#### b. 如何处理超长列表渲染
* 分片：根据数据大小，每次加载固定的数量。但有个明显缺点，加载越多，页面上累积的dom元素会很多，对性能不友好

```javascript
let total = 10000;
let index = 0;
let id = 0;
const NUM = 20
function load(){
    index += NUM;
    if(index < total){
        //也可以用setTimeout，requestAnimationFrame可以配合浏览器刷新频率
        requestAnimationFrame(()=>{
            let fragment = document.createDocumentFragment();
            //分片渲染
            for(let i = 0;i<NUM;i++){ //ie浏览器，需要用到文档碎片
                let li = document.createElement('li')
                li.innerHtml = i;
                fragment.appendChild(li)
            }
            document.getElementById('container').appendChild(li)
            load()
        })
    }
}
```

* 虚拟列表优化，只渲染当前的可视区（可以参考github上一个很成熟的vue插件：vue-virtual-scroll-list）

**原理**：把列表当成一个数组，用`开始指针`和`结束指针`去决定可视区域的数据展示，当滚动条滑动的时候，去算滑过多少个了，指针也跟着移动位置，去指向应该展示的数据位置；如果列表每项不定高的话，还需要刷新每项的高度和滚动条的高度，这部分比较复杂。

具体实现看第二部分

### 2. 超长单列表的渲染以及优化
思路大概是：
  * 不管列表多长，我需要只显示可视区域有数据
  * 告诉我的列表每一项多高(`itemH`)和多少条数据(`datas`) ==> 滚动条的高度
  * 可设定显示多少条(`showNum`)
  * 可设定是否不定高加载

1. **列表结构设计**: 
   1. 主容器：可滚动的盒子，监听滚动事件。设置相对定位和`overflow:scroll`
   2. 内层虚拟列表：设置相对定位，用于撑开主容器，用于展示数据

```html
<!-- 主容器 -->
<div class="viewport" ref="view" @scroll="handleScroll">
    <!-- 虚拟列表，用padding-top来做偏移量 -->
    <div 
        class="virtual-list"  
        ref="virtualList" 
        :style="{paddingTop:`${offset}px`,lineHeight:`${itemH}px`}"
    >
        <div v-for="(item,index) in virtualList" class="item" :key="item.key" ref="item">
        <!-- 插槽：组件通过slot-scope="{item}"接收 -->
            <slot :item="item"></slot>
        </div>
    </div>
</div>
<style lang='stylus' scoped>
.viewport
    width 500px
    position relative
    overflow-y scroll
    .virtual-list
        position relative
        top 0
        left 0
        .item
            box-sizing border-box
</style>
```
2. **初始化:`mouted阶段`**
   1. 主容器的高度 = itemH * showNum
   2. 虚拟列表的真实高度 = itemH * datas.length
   3. end = start + showNum
   4. 虚拟列表数据：`virtualList = datas.slice(start,end)`,切割真实列表来形成
   
3. **滚动条监听**

```javascript
handleScroll() {
    let scrollTop = this.$refs.view.scrollTop;
    //向下取整,滑过多少个了
    this.start = Math.floor(scrollTop / this.itemH); 
    this.end = this.start + this.showNum;
    this.offset = this.start * this.itemH; //虚拟列表视口偏移量     
},
```
到这，一个最基础的虚拟列表加载就完成了。但还有很多细节要处理。

4.  **预留占位渲染**
如果用户滑的很快，还是会出现瞬间白屏的现象，因此需要在虚拟列表**前后**加上一品列表（数量也可以自定义）。可以通过将前后指针分别前后移动，增加虚拟列表渲染范围，同时需要将虚拟列表的视口取在中间段。

<img src="/img/2.png" alt="预留渲染图" width="100%">

```javascript
//在虚拟列表前后占一品位,返回位置
prevCount() {
    //若当前开始位置大于展示条数，说明需要占位
    return Math.min(this.start, this.showNum);
},
nextCount() {
    //若到达列表末尾，则返回剩下条数
    return Math.min(this.datas.length - this.end, this.showNum);
}

virtutalList =  this.datas.slice(
                    this.start - this.prevCount, //向前一品
                    this.end + this.nextCount) //向后一品
                
//如果有预留渲染，向上移动,可以借助上图理解
offset = start * itemH - prevCount * itemH;

```
1. 优化滚动节流:借助lodash
```javascript
import throttle from 'lodash/throttle'

created(){
    this.scrollFn = throttle(this.handleScroll,200,{leading；false})
}
```
6. 不定高加载设计

。。。 待补充

### 3. 超长多级列表的渲染

常见的超长列表渲染貌似都是考虑单列表（也就是一个列表），因为工作业务上的需求，我决定优化一下，设计成可以兼容二级列表。假设传入的是Map对象，即为二级列表

1. **需要哪些变量来控制**
* `foldFlag`: 数组，值为true和false。控制展开哪个二级列表，只能展开一个二级列表（数组值只有一个true，其它为false）
* `unFoldIndex`: Number，保存当前展开的一级列表id,没有展开则为null

2. **html结构上兼容**

通过`v-if`和`v-else`来决定显示哪种列表

```html
<div class="viewport" ref="view" @scroll="scrolFn" :style="{maxWidth:setMaxWidth}">
    <!-- 虚拟列表，用paddingTop来做偏移量 -->
    <div class="virtual-list" ref="virtualList"
        :style="{paddingTop:`${offset}px`,lineHeight:`${itemH}px`}" >
        <div class="item"
            v-for="(item,index) in virtualList"
            :key="item[0]+'-'+index">
            <!-- 如果传的是数组，那就是单级列表 -->
            <div class="single-list" v-if="Array.isArray(datas)" :style="{height:`${itemH}px`}">
                <!-- 插槽：放置每一项列表，组件通过slot-scope="{item}"接收 -->
                <slot name="singleList" :item="item"></slot>
            </div>
            <!-- 传的是Map，多级列表 -->
            <div class="main-list" :style="{height:`${itemH}px`}" @click="toggleFold(index,item)"
                v-else >
                <!-- 下拉icon -->
                <span style="display:inline-block;width:12px">
                    <span
                        class="el-icon-caret-right"
                        :style="{transform: foldFlag[index]?'rotate(90deg)':'rotate(0deg)'}"
                        v-show="datas && datas.get(item[0])['children'].length>0"
                    ></span>
                </span>
                <!-- 放置一级菜单的slot -->
                <slot name="main" :main="item[0]">{{item[0]}}</slot>
            </div>
            <!-- 展示的虚拟数据，若没有二级子级，便不显示 -->
            <ul class="fold_tree"
                style="padding-left:30px;list-style: none;margin:0;"
                v-if="!Array.isArray(datas)&&foldFlag[index]&&item[1]['children'].length>0"
            >
                <li style="white-space: nowrap;"
                    v-for="(child,idx) in item[1]['children'].slice()"
                    :key="road[label]+'-'+road[nodekey]"
                >
                    <slot name="sub" :sub="child"></slot>
                </li>
            </ul>
        </div>
        <div v-if="datas.length == 0||datas.size==0">暂无数据</div>
    </div>
</div>
```

3. **初始化阶段**
   
默认不展开二级菜单，则初始化只需展示所有的一级菜单即可
* 滚动条长度 = `this.datas.size` * `this.itemH`
* start = 0 
* end = start + showNum > `this.datas.size` ? showNum :datas.size;
* 虚拟列表数据：当前没有展开，便返回整个`datas`

4. **展开二级列表设计**

思路为：点击的时候去判断：以下展开情况，并计算各种情况的真实高度
```javascript
toggleFold(index, item) {
    let scrollH;
    if (this.unFoldIndex === index) {
        //若当前点击等于自己，则收起
        this.$set(this.foldFlag, index, !this.foldFlag[index]);
        this.unFoldIndex = null;
        scrollH = this.itemH * (this.datas.size + 1);

    } else if (this.unFoldIndex == null) {
        //若是当前没有展开，则展开当前点击
        this.$set(this.foldFlag, index, true);
        //算上当前页面已有的一级列表高度
        scrollH = this.itemH * (this.datas.size +  1 +
                  this.datas.get(item[0])["children"].length);
        this.unFoldIndex = index;
    } else {
        //若点击时已有别的列表展开，便收起其它列表，展开当前列表
        this.$set(this.foldFlag, this.unFoldIndex, false);
        this.$set(this.foldFlag, index, true);
        scrollH = this.itemH *  (this.datas.size + 1 +
                  this.datas.get(item[0])["children"].length);
        this.unFoldIndex = index;
    }
    //不管展开或者收起，都将指针初始化
    this.start = 0; //控制当前列表的指针
    this.end = this.start + this.showNum;
    this.$refs.virtualList.style.height = scrollH + "px";
}
```
5. **虚拟列表的截取**

思路：先判断有无展开，无展开便展示一级菜单；有展开便去获取当前展开的一级菜单的id和二级项，然后在二级项去截取。向后占位也要判断是否有展开

```javascript
virtualList() {
    if (this.datas) {
        if (Array.isArray(this.datas)) {
            return this.datas.slice(
                this.start - this.prevCount,
                this.end + this.nextCount
            );
        } else {
            if (this.unFoldIndex === null) {
                //若当前没有展开
                return this.datas;
            } else {
                //若当前展开了，找到展开的一级菜单，切割当前列表
                //深拷贝对性能不太好
                // let data = _.cloneDeep(this.datas);
                let data = new Map();
                //优化后写法,截取当前展开的二级菜单数组
                this.datas.forEach((road, name) => {
                    let obj = { children: [], id: road.id };
                    obj[this.label] = road[this.label];
                    obj[this.nodekey] = road[this.nodekey];
                    data.set(name, obj);
                });
                // 返回截取后的Map
                let key = [...this.datas][this.unFoldIndex][0];
                let roadArr = [...this.datas][this.unFoldIndex][1]["children"].slice(
                    this.start - this.prevCount,
                    this.end + this.nextCount
                );
                //替换展现的二级数组
                let newObj = data.get(key);
                newObj["children"] = roadArr;
                data.set(key, newObj);
                return data;
            }
        }
    }
},
//向后占位也要判断是否有展开
nextCount() {
    //若到达列表末尾，则返回剩下条数
    if (Array.isArray(this.datas)) {
        return Math.min(this.datas.length - this.end, this.showNum);
    } else {
        //若当前没有展开
        if (this.unFoldIndex === null) {
            return Math.min(this.datas.size - this.end, this.showNum);
        } else {
            //若展开，找到当前展开二级菜单所有数据
            let data = [...this.datas][this.unFoldIndex][1];
            return Math.min(data.length - this.end, this.showNum);
        }
    }
}
```
6. **滚动监听**

思路：如果有展开，并且滚动距离已经超过当前要展开的一级列表上面菜单的总高度，便去更改`start`，`end`，`offset`

```javascript
handleScroll() {
    let scrollTop = this.$refs.view.scrollTop;
    if (Array.isArray(this.datas)) {
        this.start = Math.floor(scrollTop / this.itemH); //向下取整,滑过多少个了
        this.end = this.start + this.showNum;
        //如果有预留渲染，向上移动
        this.offset = this.start * this.itemH - this.prevCount * this.itemH;
    } else {
        //如果有展开，才会去改变展示数据
        if (this.unFoldIndex != null) {
            //如果滑动距离超过展开上面一级菜单的长度
            if (scrollTop > (this.unFoldIndex + 2) * this.itemH) {
                //截取的开始指针= 滑过多少个
                this.start = Math.floor(
                    (scrollTop - (this.unFoldIndex + 1) * this.itemH) / this.itemH
                );
                //虚拟列表的偏移量，应该向下顶多少
                this.offset = this.start * this.itemH - this.prevCount * this.itemH;
            } else {
                this.start = 0;
                this.offset = 0;
            }
            this.end = this.start + this.showNum;
        }
    }
},
```

7. **勾选功能的设计**

引入勾选功能，可以在加上一个是否开启可以勾选的变量。思路是：用一个集合`new Set()`来保存已选中的item,每条的勾选框的`v-model`用是否存在勾选集合中来显示。
勾选分为三种情况：全选，一级全选，二级单条勾选，因此在勾选的时候要分情况考虑

* 全选：判断当前是否勾选了->若勾选了，取消勾选，并`set.clear()`,清空集合；->若没有勾选，全选，并把所有一级菜单和二级菜单全部`add`
* 一级全选：判断当前是否勾选了-> 若勾选了，取消当前一级勾选和全选和当前子集 -> 若没有勾选，勾选一级菜单和当前子集，判断当前一级菜单是否全部勾选了（`arr.every(()=>{})`）
* 二级单条勾选：判断当前是否勾选了 -> 若勾选了，取消勾选自己和当前所在一级勾选和全选 -> 若没有勾选，勾选自己，并且判断当前所在一级菜单是否全选了和是否全选了

8. **过滤搜索功能的设计**

思路是先给数据打上拼音标识，比如网格001（wangge001），国道s108（guodaos108），然后再输入的时候，将输入值转成拼音，在所有数据的拼音标识遍历，若包含在里面便返回

```javascript
//给标签加拼音标识
addPYtag() {
    //单极列表的情况
    if (Array.isArray(this.datas)) {
        //先加载再去执行
        setTimeout(() => {
            this.datas = this.datas.map(
                e => {
                    e["py"] = this.setPinyinConvert(e[this.label]);
                    return e;
                }
            );
        }, 0);
    } else {
        setTimeout(() => {
            //遍历所有数据
            this.datas.forEach((val, key) => {
                if (val["children"].length > 0) {
                    //若有子集
                    val["children"] = val["children"].map(e => {
                        e["py"] = this.Pinyin.convertPinyin(e[this.label]);
                        return e;
                    });
                    this.datas.set(key, val);
                } else {
                    val["py"] = this.Pinyin.convertPinyin(val[this.label]);
                    this.datas.set(key, val);
                }
            });
        }, 0);
    }
}

//模糊搜索:
// @searchVal：搜索值
// @resultMapName：要保存的返回结果的变量名
// @dataName：已有拼音标识的原始数据变量名
querySearch(searchVal, resultMapName, dataName) {
    let results = [];
    //如果有搜索词
    if (searchVal) {
        // 将搜索词转成拼音
        let py = this.Pinyin.convertPinyin(searchVal.toLowerCase());
        if (!Array.isArray(this[dataName])) {
            //在py字段搜索这个拼音，符合返回
            this[dataName].forEach((val, key) => {
                if (val["children"].length > 0) {
                    results = results.concat(val["children"].filter(e => {
                        e["parentKey"] = key;
                        return (e["py"].toLowerCase().indexOf(py) > -1);
                    }));
                } else {
                    if (val["py"].toLowerCase().indexOf(py) > -1) {
                        results.push(val);
                    }
                }
            });
        } else {
            this[dataName].forEach(item => {
                if (item["py"].toLowerCase().indexOf(py) > -1) {
                    results.push(item);
                }
            });
        }
    } else {
        results = this[dataName];
    }
    this[resultMapName] = results;
}
```