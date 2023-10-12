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

---
本文以开发者视角从代码层面分析及学习HikariCp的设计思路及技术应用，深入了解一下"光"背后的秘密。   


```mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#289952',
      'primaryTextColor': '#fff',
      'primaryBorderColor': '#289952',
      'lineColor': '#289952',
      'secondaryColor': '#289952',
      'tertiaryColor': '#fff'
    }
  }
}%%
sequenceDiagram
   HikariDataSource ->> HikariPool: getConnection
   HikariPool ->> ConcurrentBag: borrow
   ConcurrentBag ->> HikariPool: poolEntry
   HikariPool ->> ProxyFactory: createProxyConnection
   ProxyFactory ->> HikariPool: getProxyConnection
   HikariPool ->> HikariDataSource: Connection
```

## HikariDataSource
```mermaid
%%{
  init: {
    'theme': 'base',
    'themeVariables': {
      'primaryColor': '#289952',
      'primaryTextColor': '#fff',
      'primaryBorderColor': '#289952',
      'lineColor': '#fff',
      'secondaryColor': '#289952',
      'tertiaryColor': '#fff'
    }
  }
}%%
classDiagram
  class HikariDataSource
  HikariDataSource : -AtomicBoolean isShutdown
  HikariDataSource : -HikariPool fastPathPool
  HikariDataSource : -HikariPool pool
  HikariDataSource : +HikariDataSource()
  HikariDataSource : +HikariDataSource(HikariConfig configuration)
  HikariDataSource : +getConnection() Connection
```

## HikariPool

## ConcurrentBag

## ProxyFactory

## 参考文档
+ [github/HikariCp](https://github.com/brettwooldridge/HikariCP)