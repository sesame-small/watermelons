---
title: HikariCp
layout: page
nav_order: 1
parent: Notes
permalink: note/hikaricp
---

# {{ page.title }}

> Fast, simple, reliable. HikariCP is a "zero-overhead" production ready JDBC connection pool. At roughly 130Kb, the library is very light. 
> 一个简单、轻量（130kb）、稳定、高性能的JDBC数据库链接池，SpringBoot2.0开始作为Spring应用的默认数据库链接池。  


### 一、HikariCp有多快
![](https://github.com/brettwooldridge/HikariCP/wiki/HikariCP-bench-2.6.0.png)  

- 每毫秒能执行47609次数据库链接的创建及关闭
- 每毫秒能执行146667次的sql statement操作 


### 二、HikariCp为什么快

### 三、核心代码分析

### 参考文档
+ [github/HikariCp](https://github.com/brettwooldridge/HikariCP)