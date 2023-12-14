---
title: Virtual Thread
layout: page
nav_order: 1
parent: Java
permalink: java/virtualthread
---

# {{ page.title }}  

--- 
> 虚拟线程是Java21引入的一种基于[Project Loom](https://github.com/openjdk/loom){:target="_blank"}技术实现的轻量级线程，是一种在用户空间中运行的线程，它们可以在操作系统级别的线程上高效地并行执行，降低了线程上下文切换的开销，极大的增强了Java语言对高并发的实现。

### 平台线程 与 虚拟线程  

| 属性 | 平台线程（Platform Thread） | 虚拟线程（Virtual Thread） |
| :--- | :--- | :--- |
| 级别 | 操作系统级别 | 用户空间级别 |
| 实现方式 | 由操作系统直接管理，一个平台线程对应一个OS线程 | 由JVM内部实现，在用户空间中运行，并可以在多个OS线程之间调度 |
| 创建成本 | 较高，涉及到操作系统的资源分配 | 较低，只需要很小的内存开销即可创建大量虚拟线程 |
| 上下文切换 | 成本较高，因为涉及到了硬件级别的上下文切换 | 成本较低，由于是在用户空间内进行，可以更高效地执行 |
| 可伸缩性 | 受到操作系统限制，数量有限 | 更好地支持高并发场景，可以创建大量虚拟线程来处理任务 |
| 适用场景 | 对性能要求极高且需要精确控制资源的应用程序 | 需要处理大量并发任务且对延迟不敏感的应用程序 |


### 参考文档：
+ [Java SE 21官网文档](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html#GUID-DC4306FC-D6C1-4BCC-AECE-48C32C1A8DAA){:target="_blank"}
+ [OpenJdk loom](https://github.com/openjdk/loom){:target="_blank"}