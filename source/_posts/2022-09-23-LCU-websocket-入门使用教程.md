---
layout: post
title: LCU websocket 入门使用教程
date: 2022-09-23
categories: 技术
tags: LCU 英雄联盟
---

除了 LCU REST API，League 客户端架构还使用 websocket 连接将更改从 LCU 本身传送到要显示给你的 UX 流程（例如，你收到的好友请求或聊天消息）。本指南将向你展示如何连接到此 websocket 并请求有关状态更改的通知。这篇文章假定你具备 LCU API 的基本知识。LCU websocket 在这个 websocket 上使用 WAMP 1.0 协议。

## 连接到Websocket

有了手中的密码和端口，这应该就像告诉 websocket 客户端/库连接到`wss://127.0.0.1:12345/`。 但要注意的是，你仍然必须像调用 LCU REST API 端点一样在请求Header中传入 authorization 字段。

如果连接成功，它应该保持打开状态而没有任何响应，如果出现任何问题，你获得的唯一信息是 HTTP 响应代码，但可能无法连接，唯一原因是你没有正确的配置Header。

## 与 websocket 通信

WAMP 1.0 对每条消息都使用 JSON 数组。数组的第一项始终是操作码。`5`为你订阅一个事件，为`6`你取消订阅一个事件，并且`8`是客户端在向你发送事件时将使用的操作码。

### 订阅事件

为了订阅一个事件，你需要发送一个第一个索引为 5 的 JSON 数组，订阅方法的操作码和一个字符串的第二个索引，告诉 LCU 你想订阅哪个事件。

例如，发送`[5, "OnJsonApiEvent"]`将使你订阅客户端发送的每个事件。如需更完整的事件列表，你可以调用`/help`客户端提供的端点，`OnJsonApiEvent`将订阅你每个可用的事件。

### 退订事件

这就像订阅一样简单，发送一个数组，第一个索引为 6，第二个索引是你要取消订阅的事件的名称，例如：`[6, "OnJsonApiEvent"]`

请记住，你不能订阅 OnJsonApiEvent 然后取消订阅你不感兴趣的较小事件。你必须首先订阅较小的事件。

### 接收更新

接收更新有点不同。数组的前 2 个索引与以前相同，8 将是操作码，第二项将是事件的名称，第三项将是具有 3 个条目的 JSON blob `data`，`eventType`和`uri`。

- uri 将告诉你更改的确切路径（对于包含标识符的聊天等事件可能很有用）
- eventType 将让你知道在该路径上发生的操作 ( `Create`, `Update`, )`Delete`
- 数据 100% 特定于事件，但它们通常遵循各自 LCU 端点使用的相同结构

这是一个例子：

[8,"OnJsonApiEvent",{"data":[],"eventType":"Update","uri":"/lol-ranked/v1/notifications"}]

请记住，如果你订阅了两个`OnJsonApiEvent`事件和相应的较小事件，你将收到两次事件，因此请小心。

## 结论

就这么多！有了这些知识，与定期轮询 LCU API 相比，你应该能够更轻松地构建响应式应用程序。