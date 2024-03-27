---
title: 前端在DDD的一些实践
date: 2023-10-10 23:43:07
tags: [DDD]
---

## Background
* 在供应链这种复杂的业务系统背景下，当时所在的Retail团队，前端开发的工作内容大部分都是开发**交互简单但业务逻辑复杂的单页面**。随着时间和功能需求的累积，面临的困境有：
  * 逻辑分散，梳理难度大，业务场景越来越复杂，**质量问题越来越多**，连测试也难以覆盖所有场景
  * 代码修改牵一发动全身，**不敢随意修改和改造**
  * 从Element-UI组件库切换到SSC-UI组件库时，**业务组件无法迁移**，只能重写
* 自身工作上的思考：
  * **只要解决好数据驱动界面的问题,我就算把代码写好了吗？（**~[关于DDD的前期调研和设想](https://confluence.shopee.io/pages/viewpage.action?pageId=914821098)
  * 如何在如此复杂的系统中，比较合理的掌握每一个技术细节背后的业务逻辑，**确保业务实现的准确**呢？
## 调研分析 - 什么样的代码设计/架构更适合当下的业务系统
⠀**前后端主流架构设计**
* 后端架构（java：j2ee、spring boot、spring cloud、SOA、微服务、**DDD**、DCI）
* 前端架构（vue、react、vuex/redux、router、flux、微前端、BFF）
DDD(Domain-Driven Design)是从实际业务出发，站在解决领域问题的角度去思考和设计系统的方法论
从后端基于DDD的实践中，优势有：**可以帮助团队合理的形成良好的业务沟通规范和业务逻辑沉淀流程**
**那么，前端需要DDD吗，前端可以DDD吗？**

## 我们需要在前端分层吗？
⠀前端主流框架便捷开发带来的坏处：
MVVC应用框架，更多的是解决前端视图层面上的问题，解决基于DOM元素如何在数据驱动模式下去进行**自动更新**
**那就会造成一个问题：开发者很难感知业务数据变动，是在什么时候触发的**
**在阅读代码的时候，就会发现是在视图层面和业务逻辑层频繁跳跃，需要不停的切换你思考的场景，给前端开发理解业务逻辑带来更大的成本**

如何在前端分层吗？
* 作为一名业务系统的前端开发,需要去关注这几个方面的事情:
* 业务模型(页面展示以及流程流转的条件控制等)
* 数据服务
* UI交互组件体系
前端要关注的范畴，应该比后端还大一些
**前端如果要实施DDD，不可能照搬后端的实践。**前端无法脱离交互去谈业务模型

## Frontend Layered Design
### 分层设计
![项目结构分层](/img/ddd/项目结构分层.png)
+ 数据访问层 DAL(Data Access Layer): 数据存储的接口，包括本地、代码、后台等数据来源
+ 业务逻辑层 BLL(Business Logic Layer): 实现业务模型和交互，按场景输出
+ 控制层 Controller: 用于连接bll和pages，配置访问路由 
+ 视图层 Pages: 各场景视图入口，统管内部所有组件
+ 基础通用层 Common: 主要是业务无关性依赖抽象工具或组件

### 业务逻辑层设计
![bll主要代码骨架](/img/ddd/bll主要代码骨架.png)
**Structure of bll：**
Store： Business Data
Schema： Business Field
eventTypes：Action Type
actions： Business Action Logic

**Method of bll：**
$options: Basic Attrs passed from View
$on: Register an event
$emit: Trigger an event
updateSchema：Update Field
updateStore： Update Store data

代码结构：
![bll主要代码骨架](/img/ddd/bll主要代码骨架.png)

### 实际业务场景使用
![具体业务场景处理](/img/ddd/具体业务场景处理.png)
### 业务逻辑层如何和视图层交互 
1. 视图层通过new BllContext() 来指定业务模块 
2. 如果有需要的话，在视图层注册视图层的事件 
3. 视图上的交互引发的业务数据变更应该调用BLL的用例

![在视图层调用bll](/img/ddd/在视图层调用bll.png)
### 延伸思考 - 落地的成本
* **上手成本和学习成本比较高**，在业务架构频繁变动的前提下，接手的时间成本与约束下带来的收益不成正比
* BLL的设计更多是服务于复杂业务系统类型的开发，**轻量级的单页面反而加重了开发的成本**
* DDD是一种方法论，目的是将业务领域内聚，**实现的核心在于如何更好服务于具体业务场景**
