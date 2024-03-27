---
title: Code Abstraction and Practical in DDD
date: 2023-10-10 23:43:07
tags: [DDD]
---

## Introduction
* Background：In the complex business system of the supply chain, the front-end development work in the Retail team primarily involved creating single-page applications with **simple interactions but complex business logic.** Over time, as functionality accumulated, the team faced the following challenges:
  * Dispersed logic, making it difficult to streamline
  * Increasing complexity of business scenarios leading to **more quality issues**, even challenging test coverage.
  * **Fear of making code changes** due to the risk of unintended consequences.
  * **Difficulty migrating business components** from Element-UI to SSC-UI when switching component libraries, often requiring a complete rewrite
+ Personal Thinking：
  * Is solving the problem of data-driven interfaces enough, or should I consider preliminary ~[research and planning related to Domain-Driven Design \(DDD\)](https://confluence.shopee.io/pages/viewpage.action?pageId=914821098)
  * In such a complex system, how can I effectively understand the business logic behind each technical detail to ensure **accurate business implementation**?
## Research and Analysis 
+ What Kind of Code Design/Architecture Is More Suitable for the Current Business System
+ **Mainstream Architecture Design for Frontend and Backend**
  * Backend Architecture (java: j2ee, spring boot, spring cloud, SOA, microservices, DDD, DCI)
  * Frontend Architecture (vue, react, vuex/redux, router, flux, micro frontends, BFF)⠀
DDD (Domain-Driven Design) is an approach that starts from real business needs, focusing on solving domain problems when thinking and designing systems.
From the practice of backend development based on DDD, the advantages include: i**t can help teams establish effective business communication standards and solidify the process of business logic.**
**So, does frontend need DDD? Can frontend implement DDD?**
## Do we need drive layering in Front-End?
The drawbacks of mainstream frontend frameworks' convenient development:
MVVC application frameworks primarily address frontend view-level issues, solving how to automatically update based on DOM elements in a data-driven model.

This creates a problem: developers find it challenging to perceive when business data changes and what triggers those changes. When reading the code, you'll notice frequent jumps between the view layer and business logic layer, requiring constant context switching, **which adds more complexity to understanding the business logic for frontend development.**

## How to Implement?
As a frontend developer in a business system, there are several aspects to focus on:
* Business Models (including conditions for page display and workflow control)
* Data Services
* UI Interaction Component System
The scope of concern for frontend should be broader than that of the backend.
**When it comes to implementing Domain-Driven Design (DDD) in the frontend, it cannot simply mimic backend practices. Frontend cannot separate itself from interactions to discuss business models.**

## Frontend Layered Design
![The Design of Business Logic Layer](/img/ddd/The Design of Business Logic Layer.png)
### Directory Structure for Code
* **view layout**
**📁 components**
**📁 pages**
* **business logic layout**
**📃 index: Entry File**
**📁 constants**
**📁 actions：Business Use Cases**
**📁 schemas：Business Use Feilds**
**📁 utils**

* Data Access Layer (DAL): Interface for data storage, including various data sources such as local, code, and backend.
* Business Logic Layer (BLL): Implements business models and interactions, delivering outputs based on specific scenarios.
* Controller: Connects BLL and pages, configures access routes.
* Pages: View Page, overseeing all internal components.
* Common: Primarily includes utils and components without business logic
### The Design of Business Logic Layer
![项目结构分层](/img/ddd/项目结构分层.png)
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

![bll主要代码骨架](/img/ddd/bll主要代码骨架.png)
![具体业务场景处理](/img/ddd/具体业务场景处理.png)
![在视图层调用bll](/img/ddd/在视图层调用bll.png)

## Elaborating on Considerations - Implementation Effort
* **The initial and learning costs are relatively high, and under the premise of frequent changes in the business architecture, the time cost of taking over does not proportionally align with the constraints imposed for benefits.**
* The design of BLL (Business Logic Layer) primarily serves the development of complex business system types, while lightweight single-page solutions actually increase development costs.
* DDD (Domain-Driven Design) is a methodology with the aim of achieving domain cohesion, and its core lies in how to better serve specific business scenarios.
