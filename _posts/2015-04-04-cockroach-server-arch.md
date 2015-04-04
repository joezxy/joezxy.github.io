---
layout: post
category: "code"
title:  "Cockroach的基本结构"
tags: [Cockroach]
---

* Cockroach的基本结构：![Cockroach Server Hierarchy](https://raw.githubusercontent.com/joezxy/joezxy.github.io/master/_img/20150404_cockroach_server_hierarchy.png)
* client.KVSender是Cockroach中所有Sender实现的接口，以此实现了类似Decorator的模式，其中有两个方法：
	* Send(call *client.Call)
	* Close()
* client.HTTPSender的功能是从客户端向服务端发送HTTP操作及管理请求
	* 发送到服务端的"/kv/db/"路径，由Server的kvDB进行处理
	* 50ms到5s之间的重试周期
	* 如果有error，将error放入call.Reply.Header中
* client.TxnSender只有在client.KV.RunTransaction时才会初始化，主要的功能为根据返回错误值设置事务的重试参数等
* client.KV是最主要的client部件
	* 在客户端使用时，使用HTTPSender进行初始化
	* 主要的外部方法有Call、Prepare、Flush、RunTransaction四个
	* 另外，GetI和GetProto均会调用getInternal获取Key的值，而GetI随后使用Gob进行解码，GetProto使用PB进行解码。PutI、PutProto和putInternal同理。PrepareProto会使用PB编码Value，在调用kv.Prepare将其放入Prepare队列中。
	* 