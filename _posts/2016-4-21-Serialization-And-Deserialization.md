---
layout: article
title: Serialization And Deserialization
---

# Serialization And Deserialization

## ------

### Jute
---

简介    
zookeeper中负责序列化的组件，比较简单也比较过时。    

结构    

* Interface
  * Index: 该接口用来实现一个遍历map或vector的Iterator
  * Record: 表明jute采用的序列化格式，要完成序列化都必须实现该接口 
  * InputArchive: 所有反序列化器都必须实现的接口
  * OutputArchive: 所有序列化器都必须实现的接口
* Class 
  * BinaryInputArchive: InputArchive的二进制实现，将二进制流反序列化成原始数据
  * BinaryOutputArchive: OutputArchive的二进制实现，将数据序列化成二进制流
