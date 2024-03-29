---
title: 理解 Redis 的持久化机制
date: 2020-04-21 18:55:17
tags: Redis
categories: 服务端开发
---

因为 Redis 是内存型数据库，它将自己的数据库状态存储在内存里面，所以如果不想办法将存储在内存中的数据库状态保存到磁盘里面，那么一旦服务器进程退出，服务器中的数据库状态也会消失不见。为了解决这个问题，Redis 提供了 RDB、AOF 持久化功能，这个功能可以将 Redis 在内存中的数据库状态保存到磁盘里面，避免数据意外丢失。
