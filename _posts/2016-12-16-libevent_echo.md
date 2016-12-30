---
title: libevent echo client 
---
# libevent echo client 

##　目标  
使用libevent使用echo的client. 编写client的时候，我们需要关心connect操作　　　
是否成功，以及如何处理stdin。

## 数据结构
对于libevent来说，每个线程有且只有一个event_base，对应一个struct event_base结构体   
（以及附于其上的事件管理器），用来调度托管给它的一系列event, 当一个事件发生后，   
event_base会在合适的时间（不一定是立即）去调用绑定在这个事件上的函数（传入一些预   
定义的参数，以及在绑定时指定的一个参数），直到这个函数执行完，再返回调度其他事件。
因此我们需要struct event_bash用于事件管理，使用struct event注册事件。
 
##　关键代码

